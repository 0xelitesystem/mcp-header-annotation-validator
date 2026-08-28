# MCP header annotation validator

Paste an MCP tool definition and find out, clause by clause, whether a conforming Streamable HTTP client will silently delete it from `tools/list` because of an `x-mcp-header` annotation.

**Live demo:** https://0xelitesystem.github.io/mcp-header-annotation-validator/

## Use

Paste the tool definition object (the one with `name` and `inputSchema`), or a bare `inputSchema`, and press Validate. Eight worked samples are on the page.

You get three things.

**A verdict, in one of three states.**

- `CONFORMS` - every annotation satisfies every constraint that is decidable from the definition alone.
- `DROPPED FROM tools/list` - at least one constraint is violated. A conforming client on the Streamable HTTP transport MUST exclude the whole tool from the result of `tools/list`. It SHOULD log a warning, client-side. Your server is told nothing.
- `SPEC-AMBIGUOUS` - nothing is clearly violated, but an annotation sits in a shape revision 2026-07-28 does not resolve. Clients may differ, so the tool may be visible in one client and gone in another.

**A clause-by-clause table**, one row per constraint in the published list: non-empty, HTTP field-name token syntax (`1*tchar`, RFC 9110 Section 5.1), no control characters including CR and LF, case-insensitive uniqueness across the entire `inputSchema`, primitive types only with `number` excluded, the IEEE754 safe-integer bound, and static reachability.

**An eligibility map**, one row per schema position below the root, saying whether it is legal to annotate there and why not. Load the "Eligibility demo (no annotations)" sample and the tool reports 14 positions, 6 of them eligible, on a schema that carries no annotations at all. Most of a real schema is not annotatable, and the map shows exactly which parts.

### What the checks actually do

**Static reachability is a graph traversal, not a regex.** An annotated property is reachable only when every hop from the schema root is a `properties` key. The traversal treats all of these as chain breakers, and names the specific one in the finding: `prefixItems`, `contains`, `additionalItems`, `unevaluatedItems`, `additionalProperties`, `patternProperties`, `propertyNames`, `dependentSchemas`, `unevaluatedProperties`, `oneOf`, `anyOf`, `allOf`, `not`, `if`, `then`, `else`, `$ref`, `$defs`, `definitions`, `contentSchema`. It still walks into the unreachable regions, because annotations there are real findings.

**The headline case is your SDK's default output, not something you typed.** Pydantic's `model_json_schema()` and `zod-to-json-schema` both emit `$defs` plus `$ref` for any nested model, and `anyOf` for any optional field. An annotation that lands inside either one has no static path from the root, and that is a rejection. Nobody wrote the `$ref`. The generator did.

**Unreferenced `$defs` still count.** A dead `$defs` block that nothing points at is both a reachability violation in its own right and a participant in the case-insensitive uniqueness scan, so it can collide with a live annotation elsewhere in the schema. The tool reports the def name in the JSON pointer path.

**Scope is `inputSchema` only.** The constraint list names `inputSchema` and nothing else. Annotations found in `outputSchema` are counted and reported as out of scope, never as violations, and they do not participate in the uniqueness scan. Flagging them would be a false positive.

**The integer bound is often undecidable, and the tool says so.** A definition holds no argument values, so the safe-integer range is only checkable when the schema statically pins a value through `const`, `enum`, `default`, `maximum` or `minimum`. Otherwise the tool prints "not determinable" instead of guessing. It also scans your raw pasted text for integer literals beyond the safe range, because every JavaScript parser rounds those before any check can see them.

**Value encoding is firewalled.** Turning a call argument into the bytes of an `Mcp-Param-{name}` field value is a runtime obligation, not a definition-validity clause. A definition holds no values, so a non-ASCII `default` is conforming and visible. The encoder panel is on the page, it computes UTF-8 bytes and standard base64 for a sample value, and it never feeds the verdict.

## Why this exists

Rejection is at MUST and it removes the entire tool, not the offending annotation. The feedback path is at SHOULD, and it is a log line on the client. That asymmetry is the whole problem: a server operator can watch a tool disappear from every client on the transport with nothing in their own logs to explain it, over a `$ref` their schema generator wrote for them.

Everything in the verdict class is decidable from the `inputSchema` alone, so it does not need a server, a network round trip, or an account. It needs a JSON parser and a correct graph walk. That fits in one HTML file with no build step, no dependency tree, no telemetry and no upload, which is what this is.

## Privacy

Everything runs in your browser. The definition you paste is parsed, traversed and rendered on your own machine. Nothing leaves it. There is no network call of any kind in this page: no CDN, no font host, no analytics, no error reporting, no fetch. The only thing written to storage is your light or dark theme choice, in `localStorage`, and the page works fine when storage is blocked.

## Run locally

```
git clone https://github.com/0xelitesystem/mcp-header-annotation-validator.git
cd mcp-header-annotation-validator
```

Open `index.html` in a browser. That is the whole thing. If you would rather serve it:

```
python -m http.server 8000
```

Then open http://localhost:8000/.

## Build

There is no build. One file, `index.html`, with the CSS and JavaScript inline. No package manager, no bundler, no dependencies, no build artifacts.

## Third-party notices

Nothing here is ported, adapted or derived from another project. No code from any other project is reproduced. The checks were implemented from the published normative constraint list for `x-mcp-header` in the Model Context Protocol specification, revision 2026-07-28 (https://modelcontextprotocol.io/specification/2026-07-28/server/tools), which the page quotes verbatim for identification and commentary and attributes on the page. The token character set enforced by the field-name check is the `token` / `tchar` production of RFC 9110 Section 5.1 (https://www.rfc-editor.org/rfc/rfc9110#section-5.1), implemented from that definition and printed in full in the tool's own output so you can check it. Behaviour was implemented from a specification, not copied.

This is an independent tool. It is not affiliated with, endorsed by, or connected to the Model Context Protocol project, Pydantic, or the authors of `zod-to-json-schema`. Those names appear because their published output shapes are what trip the constraint.

## Related

- https://github.com/0xelitesystem/mcp-tool-poisoning-scanner
- https://github.com/0xelitesystem/mcp-server-privilege-inventory
- https://github.com/0xelitesystem/json-schema-to-tool-definition

## License

MIT. Copyright (c) 2026 0xelitesystem.
