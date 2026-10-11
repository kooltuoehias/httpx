# callgraph

Static call graph for the `httpx` package, built from the Python AST (stdlib only).

```sh
python tools/callgraph/callgraph.py httpx -o graph.json --summary summary.md
python tools/callgraph/selftest.py        # fixture-based checks
```

CI: `.github/workflows/callgraph.yml` runs the self-test, builds `graph.json`,
validates it, writes a summary to the job page and uploads `out/` as the
`callgraph-<sha>` artifact.

## How calls are resolved

| `resolution` | example | how |
|---|---|---|
| `module` | `helper(x)` | name defined in the same module |
| `import` | `httpx.Client(...)`, `from ._models import Request` | import table, following re-exports and relative imports |
| `local` | `inner()` | nested function in an enclosing scope |
| `self` | `self._send_single_request(...)` | method lookup on the enclosing class and its bases |
| `super` | `super().__init__(...)` | lookup starting at the base classes |
| `type` | `response.close()` with `response: Response` | parameter/return annotations, `x = Cls(...)`, `self.attr = ...`, properties |
| `constructor` | `Headers(...)` | extra edge to the class's `__init__` (edge kind `instantiate` goes to the class) |
| `external` / `builtin` | `os.path.join`, `len` | targets outside the analysed tree |

Calls on values whose type can't be determined statically (results of
external factories, loop variables, `getattr`, callbacks) are counted as
unresolved and omitted unless `--include-unresolved` is passed. Method
calls bind to the statically known class, so `self.prepare()` in a base
class points at the base implementation, not subclass overrides.

## graph.json

```jsonc
{
  "meta":  { "generator": "...", "commit": "<sha>", "options": { ... } },
  "stats": { "classes": 87, "methods": 361, "call_sites": 1559, "resolved_ratio": 0.84, ... },
  "nodes": [
    { "id": "httpx._client.Client.send", "kind": "method", "name": "send",
      "qualname": "Client.send", "module": "httpx._client", "file": "httpx/_client.py",
      "lineno": 879, "end_lineno": 928, "parent": "httpx._client.Client", "async": false },
    { "id": "os.path.join", "kind": "external", "name": "join" }
  ],
  "edges": [
    { "source": "httpx._client.Client.send", "target": "httpx._models.Response.close",
      "kind": "call", "resolution": "type", "count": 1, "lines": [927] }
  ]
}
```

Node kinds: `module`, `class`, `function`, `method`, plus `external`,
`builtin`, `unresolved` for call targets outside the tree. Edges are
aggregated per (source, target, kind); `lines` lists every call site.

Flags: `--no-external`, `--include-builtins`, `--include-unresolved`,
`--exclude DIR`, `--root` (import root for module names).
