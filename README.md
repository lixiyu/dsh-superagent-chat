# dsh-superagent-chat

A third **Source** for [dsh-superagent](https://github.com/deepseek-ai/dsh-superagent-prototype): a plain chat entry point, next to the built-in GitHub and Feishu sources.

dsh-superagent takes work from *Sources*. A Source turns a message into a Trigger; the agent assesses the Trigger under a Policy and, if allowed, opens a Ticket and runs it in its own DSH session. Today the only Sources are GitHub (notifications, issues, pull requests, comments, CI) and Feishu (chat messages). This project adds one where the conversation happens in the panel itself.

## Status

Scaffold. Nothing runs yet.

## Why the panel, not another chat app

The point is to keep the whole pipeline — request, authorization, ticket, executor, reply — and only replace where the request comes from. A personal chat window is the cheapest source: no external CLI, no bot, no account to provision.

## How a Source plugs in

`SourceKind` is an open registry, not a closed union:

```ts
export interface SourceKind<C extends Record<string, unknown>> {
  kind: string
  schema: z.ZodType<C>
  fields: FieldSpec[]
  health: string[]
  scope?(input: { meta: Record<string, unknown> }, config: C): ScopeResult
  channel?: SourceChannel<C>
  policy?: SourcePolicyPart<Record<string, unknown> & { allowed: { id: string; name?: string }[] }>
  threadContext?: ThreadContext<C>
  // ...
}
```

Registration happens inside a plugin effect:

```ts
ctx.effect(() => ctx.orchestrator.sources.register(chatKind), 'chat: source kind')
```

A Source pushes messages with `SourceContext.receive(trigger)`; `Trigger.channel` is optional, so a chat kind may have no outbound channel. `fields` generates the Source settings card, so configuring the kind needs no panel code.

The prototype documents this in `docs/008-cookbook.md`, section "Add a Source".

## Constraints found while reading the prototype

1. **The engine is gated on setup.** `Passes.pump()` returns immediately unless `setup.configured()`, and `Deployment` requires a non-optional `repository` and `github` account. A chat-only deployment therefore needs either a configured GitHub source as a gate key, or a core change making GitHub optional.
2. **The panel has no input surface.** `Route` is a closed union and `FieldSpec` only generates settings forms; there is no composer. Inbound text is a Trigger (visible in the thread view); outbound text goes through `Outbox` to a Source channel, and the ticket view renders a channel *link*, not message bodies. A chat transcript needs its own store and renderer.
3. **The reply path is `SourceChannel.send`**, which must be idempotent by message id and must return `'gone'` only when delivery is permanently impossible.

## Plan

- [ ] `chat` SourceKind: schema, fields, health, scope, policy part
- [ ] Ingestion: accept a message and call `receive()` with a Trigger
- [ ] Delivery: `SourceChannel.send` that persists the reply for the panel
- [ ] Panel: a chat page with a composer and a transcript
- [ ] Policy: allowed senders for the chat source

## Development

The prototype loads this package into a profile by link, so a local build is picked up by restarting the service:

```sh
pnpm install && pnpm build
```

Then restart the service from the dsh-superagent checkout:

```sh
scripts/dsh-superagent restart
```

## License

MIT
