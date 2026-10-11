# 실험 원본 — MCP 서버 만드는 법: 파이썬 36줄로 Claude Code에 내 엑셀 연결 (공식 SDK v2 기준)

Mygle 에이전트가 격리된 컨테이너에서 직접 실행한 기록입니다. 경로의 `/work` 는 실험용 작업 폴더입니다.

## s1 · exit 0 · 6.4s

```
$ set -euo pipefail
echo "debian: $(cat /etc/debian_version)"
echo "node: $(node -v)"
echo "npm: $(npm -v)"
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"
echo "uv path: $(command -v uv)"
uv --version
debian: 12.15
node: v22.23.3
npm: 10.9.9
downloading uv 0.12.23 aarch64-unknown-linux-gnu
installing to /root/.local/bin
  uv
  uvx
everything's installed!

To add $HOME/.local/bin to your PATH, either restart your shell or run:

    source $HOME/.local/bin/env (sh, bash, zsh)
    source $HOME/.local/bin/env.fish (fish)
uv path: /root/.local/bin/uv
uv 0.12.23 (aarch64-unknown-linux-gnu)

```

## s2 · exit 0 · 11.7s

```
$ set -euo pipefail
export PATH="$HOME/.local/bin:$PATH"
mkdir -p /work/scripts /work/logs
cd /work
uv init --app --python 3.12 sales-mcp
cd sales-mcp
uv add mcp openpyxl
uv run python -c "import sys, importlib.metadata as m; print('python', sys.version.split()[0]); print('mcp', m.version('mcp')); print('openpyxl', m.version('openpyxl'))"
cat pyproject.toml
Initialized project `sales-mcp` at `/work/sales-mcp`
Downloading cpython-3.12.15-linux-aarch64-gnu (download) (27.9MiB)
 Downloaded cpython-3.12.15-linux-aarch64-gnu (download)
Using CPython 3.12.15
Creating virtual environment at: .venv
Resolved 33 packages in 678ms
   Building sales-mcp @ file:///work/sales-mcp
      Built sales-mcp @ file:///work/sales-mcp
Downloading cryptography (4.5MiB)
Downloading pydantic-core (1.9MiB)
 Downloaded pydantic-core
 Downloaded cryptography
Prepared 31 packages in 1.23s
warning: Failed to hardlink files; falling back to full copy. This may lead to degraded performance.
         If the cache and target directories are on different filesystems, hardlinking may not be supported.
         If this is intentional, set `export UV_LINK_MODE=copy` or use `--link-mode=copy` to suppress this warning.
Installed 31 packages in 663ms
 + annotated-types==0.8.0
 + anyio==4.15.1
 + attrs==26.1.0
 + cffi==2.1.1
 + click==8.5.0
 + cryptography==50.0.2
 + et-xmlfile==2.0.0
 + h11==0.16.0
 + httpcore2==2.13.1
 + httpx2==2.13.1
 + idna==3.20
 + jsonschema==4.26.0
 + jsonschema-specifications==2025.9.1
 + mcp==2.3.0
 + mcp-types==2.3.0
 + openpyxl==3.1.5
 + opentelemetry-api==1.45.1
 + pycparser==3.0
 + pydantic==2.13.5
 + pydantic-core==2.46.5
 + pyjwt==2.15.1
 + python-multipart==0.0.32
 + referencing==0.37.0
 + rpds-py==2026.9.1
 + sales-mcp==0.1.0 (from file:///work/sales-mcp)
 + sse-starlette==3.5.0
 + starlette==1.7.0
 + truststore==0.10.4
 + typing-extensions==4.16.0
 + typing-inspection==0.4.4
 + uvicorn==0.54.0
python 3.12.15
mcp 2.3.0
openpyxl 3.1.5
[project]
name = "sales-mcp"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "mcp>=2.3.0",
    "openpyxl>=3.1.5",
]

[project.scripts]
sales-mcp = "sales_mcp:main"

[build-system]
requires = ["uv_build>=0.12.23,<0.13.0"]
build-backend = "uv_build"

```

## s3 · exit 0 · 3.2s

```
$ set -euo pipefail
export PATH="$HOME/.local/bin:$PATH"
cat > /work/scripts/check_imports.py <<'EOF'
import importlib
import importlib.metadata as m

print("mcp version:", m.version("mcp"))
for mod, name in [("mcp.server.fastmcp", "FastMCP"), ("mcp.server.mcpserver", "MCPServer")]:
    try:
        getattr(importlib.import_module(mod), name)
        print(f"OK   from {mod} import {name}")
    except Exception as e:
        print(f"FAIL from {mod} import {name} -> {type(e).__name__}: {e}")
EOF
echo '=== [v2] 최신 mcp (sales-mcp 프로젝트) ==='
cd /work/sales-mcp && uv run python /work/scripts/check_imports.py
echo '=== [v1] mcp<2 고정 환경 ==='
mkdir -p /work/v1test && cd /work/v1test
uv venv -q --python 3.12
uv pip install -q "mcp<2"
.venv/bin/python /work/scripts/check_imports.py
=== [v2] 최신 mcp (sales-mcp 프로젝트) ===
mcp version: 2.3.0
FAIL from mcp.server.fastmcp import FastMCP -> ModuleNotFoundError: No module named 'mcp.server.fastmcp'. This is mcp 2.x, where FastMCP was renamed to MCPServer (from mcp.server.mcpserver import MCPServer) and other APIs changed; see the migration guide at https://py.sdk.modelcontextprotocol.io/v2/migration/#fastmcp-renamed-to-mcpserver or pin 'mcp<2' to keep running v1 code.
OK   from mcp.server.mcpserver import MCPServer
=== [v1] mcp<2 고정 환경 ===
mcp version: 1.30.0
OK   from mcp.server.fastmcp import FastMCP
FAIL from mcp.server.mcpserver import MCPServer -> ModuleNotFoundError: No module named 'mcp.server.mcpserver'

```

## s4 · exit 0 · 0.5s

```
$ set -euo pipefail
export PATH="$HOME/.local/bin:$PATH"
cat > /work/scripts/make_data.py <<'EOF'
import random
from collections import defaultdict
from openpyxl import Workbook, load_workbook

random.seed(42)
products = {"무선 이어폰": 89000, "보조배터리": 32000, "USB-C 케이블": 12000, "노트북 거치대": 45000, "기계식 키보드": 129000}
regions = ["서울", "경기", "부산", "대구", "광주"]
wb = Workbook()
ws = wb.active
ws.title = "sales"
ws.append(["date", "region", "product", "qty", "unit_price", "amount"])
for month in ["2026-07", "2026-08", "2026-09"]:
    for _ in range(40):
        p = random.choice(list(products))
        q = random.randint(1, 5)
        ws.append([f"{month}-{random.randint(1, 28):02d}", random.choice(regions), p, q, products[p], q * products[p]])
path = "/work/sales-mcp/sales.xlsx"
wb.save(path)

rows = list(load_workbook(path, read_only=True).active.iter_rows(values_only=True))[1:]
print("data rows:", len(rows))
for month in ["2026-07", "2026-08", "2026-09"]:
    mr = [r for r in rows if r[0].startswith(month)]
    by_p = defaultdict(int)
    for r in mr:
        by_p[r[2]] += r[5]
    top = sorted(by_p.items(), key=lambda kv: kv[1], reverse=True)[:3]
    print(f"[정답] {month} orders={len(mr)} total={sum(r[5] for r in mr)} top3={top}")
EOF
cd /work/sales-mcp
uv run python /work/scripts/make_data.py
ls -l /work/sales-mcp/sales.xlsx
data rows: 120
[정답] 2026-07 orders=40 total=6730000 top3=[('기계식 키보드', 3225000), ('노트북 거치대', 1350000), ('무선 이어폰', 1335000)]
[정답] 2026-08 orders=40 total=5716000 top3=[('기계식 키보드', 2193000), ('보조배터리', 1472000), ('무선 이어폰', 1157000)]
[정답] 2026-09 orders=40 total=7284000 top3=[('기계식 키보드', 2967000), ('무선 이어폰', 2314000), ('노트북 거치대', 1035000)]
-rw-r--r--. 1 root root 8422 Oct  7 07:02 /work/sales-mcp/sales.xlsx

```

## s5 · exit 1 · 0.1s

```
$ set -euo pipefail
cd /work/sales-mcp
cat > server.py <<'EOF'
import sys
from collections import defaultdict
from pathlib import Path
from openpyxl import load_workbook
from mcp.server.mcpserver import MCPServer

DATA = Path(__file__).resolve().parent / "sales.xlsx"
mcp = MCPServer("sales")

def load_rows(month: str) -> list[dict]:
    ws = load_workbook(DATA, read_only=True).active
    rows = ws.iter_rows(values_only=True)
    header = next(rows)
    return [dict(zip(header, r)) for r in rows if str(r[0]).startswith(month)]

@mcp.tool()
def sales_summary(month: str) -> dict:
    """월별 매출 요약(총매출, 주문 수, 지역별 매출). month 는 'YYYY-MM' 형식 (예: 2026-09)."""
    rows, by_region = load_rows(month), defaultdict(int)
    for r in rows:
        by_region[r["region"]] += r["amount"]
    return {"month": month, "orders": len(rows), "total": sum(by_region.values()), "by_region": dict(by_region)}

@mcp.tool()
def top_products(month: str, n: int = 3) -> list[dict]:
    """해당 월 매출 상위 n개 상품과 판매 수량, 매출."""
    agg = defaultdict(lambda: {"qty": 0, "amount": 0})
    for r in load_rows(month):
        agg[r["product"]]["qty"] += r["qty"]
        agg[r["product"]]["amount"] += r["amount"]
    ranked = sorted(agg.items(), key=lambda kv: kv[1]["amount"], reverse=True)
    return [{"product": p, **v} for p, v in ranked[:n]]

if __name__ == "__main__":
    print("sales MCP server starting", file=sys.stderr)
    mcp.run()
EOF
sed -e 's/, file=sys.stderr//' -e 's/^    ws = load_workbook(DATA/    print("DEBUG loading", DATA)\n    ws = load_workbook(DATA/' server.py > bad_print.py
sed -e 's|Path(__file__).resolve().parent / "sales.xlsx"|Path("sales.xlsx")|' server.py > bad_relpath.py
echo "server.py total lines: $(wc -l < server.py)"
echo "server.py non-blank lines: $(grep -cv '^[[:space:]]*$' server.py)"
cat -n server.py
echo '=== diff server.py bad_print.py ==='
diff server.py bad_print.py > /work/logs/diff_print.txt; echo "diff exit: $?"; cat /work/logs/diff_print.txt
echo '=== diff server.py bad_relpath.py ==='
diff server.py bad_relpath.py > /work/logs/diff_relpath.txt; echo "diff exit: $?"; cat /work/logs/diff_relpath.txt
server.py total lines: 36
server.py non-blank lines: 31
     1	import sys
     2	from collections import defaultdict
     3	from pathlib import Path
     4	from openpyxl import load_workbook
     5	from mcp.server.mcpserver import MCPServer
     6	
     7	DATA = Path(__file__).resolve().parent / "sales.xlsx"
     8	mcp = MCPServer("sales")
     9	
    10	def load_rows(month: str) -> list[dict]:
    11	    ws = load_workbook(DATA, read_only=True).active
    12	    rows = ws.iter_rows(values_only=True)
    13	    header = next(rows)
    14	    return [dict(zip(header, r)) for r in rows if str(r[0]).startswith(month)]
    15	
    16	@mcp.tool()
    17	def sales_summary(month: str) -> dict:
    18	    """월별 매출 요약(총매출, 주문 수, 지역별 매출). month 는 'YYYY-MM' 형식 (예: 2026-09)."""
    19	    rows, by_region = load_rows(month), defaultdict(int)
    20	    for r in rows:
    21	        by_region[r["region"]] += r["amount"]
    22	    return {"month": month, "orders": len(rows), "total": sum(by_region.values()), "by_region": dict(by_region)}
    23	
    24	@mcp.tool()
    25	def top_products(month: str, n: int = 3) -> list[dict]:
    26	    """해당 월 매출 상위 n개 상품과 판매 수량, 매출."""
    27	    agg = defaultdict(lambda: {"qty": 0, "amount": 0})
    28	    for r in load_rows(month):
    29	        agg[r["product"]]["qty"] += r["qty"]
    30	        agg[r["product"]]["amount"] += r["amount"]
    31	    ranked = sorted(agg.items(), key=lambda kv: kv[1]["amount"], reverse=True)
    32	    return [{"product": p, **v} for p, v in ranked[:n]]
    33	
    34	if __name__ == "__main__":
    35	    print("sales MCP server starting", file=sys.stderr)
    36	    mcp.run()
=== diff server.py bad_print.py ===

```

## r1-s5 · exit 0 · 0.3s

```
$ set -euo pipefail
export PATH="$HOME/.local/bin:$PATH"
cd /work/sales-mcp
cat > server.py <<'EOF'
import sys
from collections import defaultdict
from pathlib import Path
from openpyxl import load_workbook
from mcp.server.mcpserver import MCPServer

DATA = Path(__file__).resolve().parent / "sales.xlsx"
mcp = MCPServer("sales")

def load_rows(month: str) -> list[dict]:
    ws = load_workbook(DATA, read_only=True).active
    rows = ws.iter_rows(values_only=True)
    header = next(rows)
    return [dict(zip(header, r)) for r in rows if str(r[0]).startswith(month)]

@mcp.tool()
def sales_summary(month: str) -> dict:
    """월별 매출 요약(총매출, 주문 수, 지역별 매출). month 는 'YYYY-MM' 형식 (예: 2026-09)."""
    rows, by_region = load_rows(month), defaultdict(int)
    for r in rows:
        by_region[r["region"]] += r["amount"]
    return {"month": month, "orders": len(rows), "total": sum(by_region.values()), "by_region": dict(by_region)}

@mcp.tool()
def top_products(month: str, n: int = 3) -> list[dict]:
    """해당 월 매출 상위 n개 상품과 판매 수량, 매출."""
    agg = defaultdict(lambda: {"qty": 0, "amount": 0})
    for r in load_rows(month):
        agg[r["product"]]["qty"] += r["qty"]
        agg[r["product"]]["amount"] += r["amount"]
    ranked = sorted(agg.items(), key=lambda kv: kv[1]["amount"], reverse=True)
    return [{"product": p, **v} for p, v in ranked[:n]]

if __name__ == "__main__":
    print("sales MCP server starting", file=sys.stderr)
    mcp.run()
EOF
sed -e 's/, file=sys.stderr//' -e 's/^    ws = load_workbook(DATA/    print("DEBUG loading", DATA)\n    ws = load_workbook(DATA/' server.py > bad_print.py
sed -e 's|Path(__file__).resolve().parent / "sales.xlsx"|Path("sales.xlsx")|' server.py > bad_relpath.py
echo "server.py total lines: $(wc -l < server.py)"
echo "server.py non-blank lines: $(grep -cv '^[[:space:]]*$' server.py)"
cat -n server.py
# diff 는 차이가 있으면 exit 1 → set -e 에서 죽지 않도록 종료코드를 따로 받는다
echo '=== diff server.py bad_print.py ==='
rc=0; diff server.py bad_print.py > /work/logs/diff_print.txt || rc=$?
echo "diff exit: $rc (1 = 차이 있음, 정상)"; cat /work/logs/diff_print.txt
[ "$rc" -eq 1 ] || { echo 'bad_print.py 변형 실패'; exit 1; }
echo '=== diff server.py bad_relpath.py ==='
rc=0; diff server.py bad_relpath.py > /work/logs/diff_relpath.txt || rc=$?
echo "diff exit: $rc (1 = 차이 있음, 정상)"; cat /work/logs/diff_relpath.txt
[ "$rc" -eq 1 ] || { echo 'bad_relpath.py 변형 실패'; exit 1; }
echo '=== 문법 검사 ==='
uv run python -m py_compile server.py bad_print.py bad_relpath.py && echo 'py_compile OK (3 files)'
server.py total lines: 36
server.py non-blank lines: 31
     1	import sys
     2	from collections import defaultdict
     3	from pathlib import Path
     4	from openpyxl import load_workbook
     5	from mcp.server.mcpserver import MCPServer
     6	
     7	DATA = Path(__file__).resolve().parent / "sales.xlsx"
     8	mcp = MCPServer("sales")
     9	
    10	def load_rows(month: str) -> list[dict]:
    11	    ws = load_workbook(DATA, read_only=True).active
    12	    rows = ws.iter_rows(values_only=True)
    13	    header = next(rows)
    14	    return [dict(zip(header, r)) for r in rows if str(r[0]).startswith(month)]
    15	
    16	@mcp.tool()
    17	def sales_summary(month: str) -> dict:
    18	    """월별 매출 요약(총매출, 주문 수, 지역별 매출). month 는 'YYYY-MM' 형식 (예: 2026-09)."""
    19	    rows, by_region = load_rows(month), defaultdict(int)
    20	    for r in rows:
    21	        by_region[r["region"]] += r["amount"]
    22	    return {"month": month, "orders": len(rows), "total": sum(by_region.values()), "by_region": dict(by_region)}
    23	
    24	@mcp.tool()
    25	def top_products(month: str, n: int = 3) -> list[dict]:
    26	    """해당 월 매출 상위 n개 상품과 판매 수량, 매출."""
    27	    agg = defaultdict(lambda: {"qty": 0, "amount": 0})
    28	    for r in load_rows(month):
    29	        agg[r["product"]]["qty"] += r["qty"]
    30	        agg[r["product"]]["amount"] += r["amount"]
    31	    ranked = sorted(agg.items(), key=lambda kv: kv[1]["amount"], reverse=True)
    32	    return [{"product": p, **v} for p, v in ranked[:n]]
    33	
    34	if __name__ == "__main__":
    35	    print("sales MCP server starting", file=sys.stderr)
    36	    mcp.run()
=== diff server.py bad_print.py ===
diff exit: 1 (1 = 차이 있음, 정상)
10a11
>     print("DEBUG loading", DATA)
35c36
<     print("sales MCP server starting", file=sys.stderr)
---
>     print("sales MCP server starting")
=== diff server.py bad_relpath.py ===
diff exit: 1 (1 = 차이 있음, 정상)
7c7
< DATA = Path(__file__).resolve().parent / "sales.xlsx"
---
> DATA = Path("sales.xlsx")
=== 문법 검사 ===
py_compile OK (3 files)

```

## r1-s6 · exit 0 · 0.1s

```
$ set -euo pipefail
export PATH="$HOME/.local/bin:$PATH"
cat > /work/scripts/make_configs.py <<'EOF'
import json, shutil
uv = shutil.which("uv")
sales = {"command": uv, "args": ["run", "--directory", "/work/sales-mcp", "server.py"]}
desktop = {"mcpServers": {"sales": sales}}
lab = {"mcpServers": {
    "sales": sales,
    "sales-bare-uv": {"command": "uv", "args": ["run", "--directory", "/work/sales-mcp", "server.py"]},
    "sales-bad-print": {"command": uv, "args": ["run", "--directory", "/work/sales-mcp", "bad_print.py"]},
    "sales-bad-relpath": {"command": uv, "args": ["run", "--project", "/work/sales-mcp", "python", "/work/sales-mcp/bad_relpath.py"]},
}}
json.dump(desktop, open("/work/claude_desktop_config.json", "w"), indent=2, ensure_ascii=False)
json.dump(lab, open("/work/inspector-lab.json", "w"), indent=2, ensure_ascii=False)
EOF
python3 /work/scripts/make_configs.py
echo '=== /work/claude_desktop_config.json (Claude Desktop 에 넣을 설정) ==='
cat /work/claude_desktop_config.json
echo
echo '=== /work/inspector-lab.json (실패 재현용 포함) ==='
cat /work/inspector-lab.json
=== /work/claude_desktop_config.json (Claude Desktop 에 넣을 설정) ===
{
  "mcpServers": {
    "sales": {
      "command": "/root/.local/bin/uv",
      "args": [
        "run",
        "--directory",
        "/work/sales-mcp",
        "server.py"
      ]
    }
  }
}
=== /work/inspector-lab.json (실패 재현용 포함) ===
{
  "mcpServers": {
    "sales": {
      "command": "/root/.local/bin/uv",
      "args": [
        "run",
        "--directory",
        "/work/sales-mcp",
        "server.py"
      ]
    },
    "sales-bare-uv": {
      "command": "uv",
      "args": [
        "run",
        "--directory",
        "/work/sales-mcp",
        "server.py"
      ]
    },
    "sales-bad-print": {
      "command": "/root/.local/bin/uv",
      "args": [
        "run",
        "--directory",
        "/work/sales-mcp",
        "bad_print.py"
      ]
    },
    "sales-bad-relpath": {
      "command": "/root/.local/bin/uv",
      "args": [
        "run",
        "--project",
        "/work/sales-mcp",
        "python",
        "/work/sales-mcp/bad_relpath.py"
      ]
    }
  }
}
```

## r1-s7 · exit 0 · 21.9s

```
$ set -euo pipefail
export PATH="$HOME/.local/bin:$PATH"
echo "inspector latest on npm: $(npm view @modelcontextprotocol/inspector version)"
cd /tmp
npx -y @modelcontextprotocol/inspector --cli --config /work/claude_desktop_config.json --server sales --method tools/list > /work/logs/tools_list.json
echo '=== tools/list (Claude Desktop 설정 그대로 사용) ==='
cat /work/logs/tools_list.json
npm notice
npm notice New major version of npm available! 10.9.9 -> 12.2.0
npm notice Changelog: https://github.com/npm/cli/releases/tag/v12.2.0
npm notice To update run: npm install -g npm@12.2.0
npm notice
inspector latest on npm: 2.9.0

[mcp-inspector] The OS keychain is not available, so secrets will be kept in: Memory (this session only) — fell back from the OS keychain.
[mcp-inspector] Keychain error: Couldn't access platform storage: AccessDenied

Caused by:
    AccessDenied
[mcp-inspector] Set MCP_INSPECTOR_SECRET_STORE=keyring|file|memory to choose explicitly.
[mcp-inspector] Secrets are not written anywhere and are lost on exit.
[mcp-inspector] How the secret store is chosen, and how to secure it: https://github.com/modelcontextprotocol/inspector/blob/main/docs/secret-storage.md
sales MCP server starting
=== tools/list (Claude Desktop 설정 그대로 사용) ===
{
  "tools": [
    {
      "name": "sales_summary",
      "description": "월별 매출 요약(총매출, 주문 수, 지역별 매출). month 는 'YYYY-MM' 형식 (예: 2026-09).",
      "inputSchema": {
        "type": "object",
        "properties": {
          "month": {
            "title": "Month",
            "type": "string"
          }
        },
        "required": [
          "month"
        ],
        "title": "sales_summaryArguments"
      }
    },
    {
      "name": "top_products",
      "description": "해당 월 매출 상위 n개 상품과 판매 수량, 매출.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "month": {
            "title": "Month",
            "type": "string"
          },
          "n": {
            "default": 3,
            "title": "N",
            "type": "integer"
          }
        },
        "required": [
          "month"
        ],
        "title": "top_productsArguments"
      },
      "outputSchema": {
        "type": "object",
        "properties": {
          "result": {
            "items": {
              "additionalProperties": true,
              "type": "object"
            },
            "title": "Result",
            "type": "array"
          }
        },
        "required": [
          "result"
        ],
        "title": "top_productsOutput"
      }
    }
  ]
}

```

## r1-s8 · exit 0 · 2.6s

```
$ set -euo pipefail
export PATH="$HOME/.local/bin:$PATH"
cd /tmp
echo '=== tools/call sales_summary month=2026-09 ==='
time npx -y @modelcontextprotocol/inspector --cli --config /work/claude_desktop_config.json --server sales --method tools/call --tool-name sales_summary --tool-arg month=2026-09
echo '=== tools/call top_products month=2026-09 n=3 ==='
time npx -y @modelcontextprotocol/inspector --cli --config /work/claude_desktop_config.json --server sales --method tools/call --tool-name top_products --tool-arg month=2026-09 --tool-arg n=3
echo '=== 정답지 다시 출력 (비교용) ==='
cd /work/sales-mcp && uv run python /work/scripts/make_data.py
=== tools/call sales_summary month=2026-09 ===

[mcp-inspector] The OS keychain is not available, so secrets will be kept in: Memory (this session only) — fell back from the OS keychain.
[mcp-inspector] Keychain error: Couldn't access platform storage: AccessDenied

Caused by:
    AccessDenied
[mcp-inspector] Set MCP_INSPECTOR_SECRET_STORE=keyring|file|memory to choose explicitly.
[mcp-inspector] Secrets are not written anywhere and are lost on exit.
[mcp-inspector] How the secret store is chosen, and how to secure it: https://github.com/modelcontextprotocol/inspector/blob/main/docs/secret-storage.md
sales MCP server starting
{
  "content": [
    {
      "type": "text",
      "text": "{\n  \"month\": \"2026-09\",\n  \"orders\": 40,\n  \"total\": 7284000,\n  \"by_region\": {\n    \"서울\": 2068000,\n    \"광주\": 1255000,\n    \"부산\": 2130000,\n    \"경기\": 892000,\n    \"대구\": 939000\n  }\n}"
    }
  ],
  "isError": false
}

real	0m1.276s
user	0m0.756s
sys	0m0.136s
=== tools/call top_products month=2026-09 n=3 ===

[mcp-inspector] The OS keychain is not available, so secrets will be kept in: Memory (this session only) — fell back from the OS keychain.
[mcp-inspector] Keychain error: Couldn't access platform storage: AccessDenied

Caused by:
    AccessDenied
[mcp-inspector] Set MCP_INSPECTOR_SECRET_STORE=keyring|file|memory to choose explicitly.
[mcp-inspector] Secrets are not written anywhere and are lost on exit.
[mcp-inspector] How the secret store is chosen, and how to secure it: https://github.com/modelcontextprotocol/inspector/blob/main/docs/secret-storage.md
sales MCP server starting
{
  "content": [
    {
      "type": "text",
      "text": "{\n  \"product\": \"기계식 키보드\",\n  \"qty\": 23,\n  \"amount\": 2967000\n}"
    },
    {
      "type": "text",
      "text": "{\n  \"product\": \"무선 이어폰\",\n  \"qty\": 26,\n  \"amount\": 2314000\n}"
    },
    {
      "type": "text",
      "text": "{\n  \"product\": \"노트북 거치대\",\n  \"qty\": 23,\n  \"amount\": 1035000\n}"
    }
  ],
  "structuredContent": {
    "result": [
      {
        "product": "기계식 키보드",
        "qty": 23,
        "amount": 2967000
      },
      {
        "product": "무선 이어폰",
        "qty": 26,
        "amount": 2314000
      },
      {
        "product": "노트북 거치대",
        "qty": 23,
        "amount": 1035000
      }
    ]
  },
  "isError": false
}

real	0m1.063s
user	0m0.726s
sys	0m0.122s
=== 정답지 다시 출력 (비교용) ===
data rows: 120
[정답] 2026-07 orders=40 total=6730000 top3=[('기계식 키보드', 3225000), ('노트북 거치대', 1350000), ('무선 이어폰', 1335000)]
[정답] 2026-08 orders=40 total=5716000 top3=[('기계식 키보드', 2193000), ('보조배터리', 1472000), ('무선 이어폰', 1157000)]
[정답] 2026-09 orders=40 total=7284000 top3=[('기계식 키보드', 2967000), ('무선 이어폰', 2314000), ('노트북 거치대', 1035000)]

```

## r1-s9 · exit 0 · 2.1s

```
$ export PATH="$HOME/.local/bin:$PATH"
cd /tmp
echo '=== [실패 재현 1] print()가 stdout 오염: tools/list ==='
npx -y @modelcontextprotocol/inspector --cli --config /work/inspector-lab.json --server sales-bad-print --method tools/list > /work/logs/bad_print_list.txt 2>&1
echo "exit code: $?"
cat /work/logs/bad_print_list.txt
echo '=== [실패 재현 1] print()가 stdout 오염: tools/call ==='
npx -y @modelcontextprotocol/inspector --cli --config /work/inspector-lab.json --server sales-bad-print --method tools/call --tool-name sales_summary --tool-arg month=2026-09 > /work/logs/bad_print_call.txt 2>&1
echo "exit code: $?"
cat /work/logs/bad_print_call.txt
=== [실패 재현 1] print()가 stdout 오염: tools/list ===
exit code: 0

[mcp-inspector] The OS keychain is not available, so secrets will be kept in: Memory (this session only) — fell back from the OS keychain.
[mcp-inspector] Keychain error: Couldn't access platform storage: AccessDenied

Caused by:
    AccessDenied
[mcp-inspector] Set MCP_INSPECTOR_SECRET_STORE=keyring|file|memory to choose explicitly.
[mcp-inspector] Secrets are not written anywhere and are lost on exit.
[mcp-inspector] How the secret store is chosen, and how to secure it: https://github.com/modelcontextprotocol/inspector/blob/main/docs/secret-storage.md
{
  "tools": [
    {
      "name": "sales_summary",
      "description": "월별 매출 요약(총매출, 주문 수, 지역별 매출). month 는 'YYYY-MM' 형식 (예: 2026-09).",
      "inputSchema": {
        "type": "object",
        "properties": {
          "month": {
            "title": "Month",
            "type": "string"
          }
        },
        "required": [
          "month"
        ],
        "title": "sales_summaryArguments"
      }
    },
    {
      "name": "top_products",
      "description": "해당 월 매출 상위 n개 상품과 판매 수량, 매출.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "month": {
            "title": "Month",
            "type": "string"
          },
          "n": {
            "default": 3,
            "title": "N",
            "type": "integer"
          }
        },
        "required": [
          "month"
        ],
        "title": "top_productsArguments"
      },
      "outputSchema": {
        "type": "object",
        "properties": {
          "result": {
            "items": {
              "additionalProperties": true,
              "type": "object"
            },
            "title": "Result",
            "type": "array"
          }
        },
        "required": [
          "result"
        ],
        "title": "top_productsOutput"
      }
    }
  ]
}
=== [실패 재현 1] print()가 stdout 오염: tools/call ===
exit code: 0

[mcp-inspector] The OS keychain is not available, so secrets will be kept in: Memory (this session only) — fell back from the OS keychain.
[mcp-inspector] Keychain error: Couldn't access platform storage: AccessDenied

Caused by:
    AccessDenied
[mcp-inspector] Set MCP_INSPECTOR_SECRET_STORE=keyring|file|memory to choose explicitly.
[mcp-inspector] Secrets are not written anywhere and are lost on exit.
[mcp-inspector] How the secret store is chosen, and how to secure it: https://github.com/modelcontextprotocol/inspector/blob/main/docs/secret-storage.md
{
  "content": [
    {
      "type": "text",
      "text": "{\n  \"month\": \"2026-09\",\n  \"orders\": 40,\n  \"total\": 7284000,\n  \"by_region\": {\n    \"서울\": 2068000,\n    \"광주\": 1255000,\n    \"부산\": 2130000,\n    \"경기\": 892000,\n    \"대구\": 939000\n  }\n}"
    }
  ],
  "isError": false
}

```

## r1-s10 · exit 0 · 2.2s

```
$ export PATH="$HOME/.local/bin:$PATH"
echo '=== [실패 재현 2] 상대경로 Path("sales.xlsx") — cwd=/work/sales-mcp 에서 실행 ==='
cd /work/sales-mcp
npx -y @modelcontextprotocol/inspector --cli --config /work/inspector-lab.json --server sales-bad-relpath --method tools/call --tool-name sales_summary --tool-arg month=2026-09 > /work/logs/relpath_devdir.txt 2>&1
echo "exit code: $?"
cat /work/logs/relpath_devdir.txt
echo '=== [실패 재현 2] 같은 서버 — cwd=/tmp 에서 실행 (클라이언트가 다른 폴더에서 띄우는 상황) ==='
cd /tmp
npx -y @modelcontextprotocol/inspector --cli --config /work/inspector-lab.json --server sales-bad-relpath --method tools/call --tool-name sales_summary --tool-arg month=2026-09 > /work/logs/relpath_tmp.txt 2>&1
echo "exit code: $?"
cat /work/logs/relpath_tmp.txt
=== [실패 재현 2] 상대경로 Path("sales.xlsx") — cwd=/work/sales-mcp 에서 실행 ===
exit code: 0

[mcp-inspector] The OS keychain is not available, so secrets will be kept in: Memory (this session only) — fell back from the OS keychain.
[mcp-inspector] Keychain error: Couldn't access platform storage: AccessDenied

Caused by:
    AccessDenied
[mcp-inspector] Set MCP_INSPECTOR_SECRET_STORE=keyring|file|memory to choose explicitly.
[mcp-inspector] Secrets are not written anywhere and are lost on exit.
[mcp-inspector] How the secret store is chosen, and how to secure it: https://github.com/modelcontextprotocol/inspector/blob/main/docs/secret-storage.md
sales MCP server starting
{
  "content": [
    {
      "type": "text",
      "text": "{\n  \"month\": \"2026-09\",\n  \"orders\": 40,\n  \"total\": 7284000,\n  \"by_region\": {\n    \"서울\": 2068000,\n    \"광주\": 1255000,\n    \"부산\": 2130000,\n    \"경기\": 892000,\n    \"대구\": 939000\n  }\n}"
    }
  ],
  "isError": false
}
=== [실패 재현 2] 같은 서버 — cwd=/tmp 에서 실행 (클라이언트가 다른 폴더에서 띄우는 상황) ===
exit code: 5

[mcp-inspector] The OS keychain is not available, so secrets will be kept in: Memory (this session only) — fell back from the OS keychain.
[mcp-inspector] Keychain error: Couldn't access platform storage: AccessDenied

Caused by:
    AccessDenied
[mcp-inspector] Set MCP_INSPECTOR_SECRET_STORE=keyring|file|memory to choose explicitly.
[mcp-inspector] Secrets are not written anywhere and are lost on exit.
[mcp-inspector] How the secret store is chosen, and how to secure it: https://github.com/modelcontextprotocol/inspector/blob/main/docs/secret-storage.md
sales MCP server starting
Tool 'sales_summary' raised an unexpected exception
Traceback (most recent call last):
  File "/work/sales-mcp/.venv/lib/python3.12/site-packages/mcp/server/mcpserver/tools/base.py", line 179, in run
    result = await self.fn_metadata.call_fn(self.fn, self.is_async, validated, pass_directly)
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/work/sales-mcp/.venv/lib/python3.12/site-packages/mcp/server/mcpserver/utilities/func_metadata.py", line 164, in call_fn
    return await anyio.to_thread.run_sync(functools.partial(fn, **kwargs))
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/work/sales-mcp/.venv/lib/python3.12/site-packages/anyio/to_thread.py", line 65, in run_sync
    return await get_async_backend().run_sync_in_worker_thread(
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/work/sales-mcp/.venv/lib/python3.12/site-packages/anyio/_backends/_asyncio.py", line 2706, in run_sync_in_worker_thread
    return await future
           ^^^^^^^^^^^^
  File "/work/sales-mcp/.venv/lib/python3.12/site-packages/anyio/_backends/_asyncio.py", line 1100, in run
    result = context.run(func, *args)
             ^^^^^^^^^^^^^^^^^^^^^^^^
  File "/work/sales-mcp/bad_relpath.py", line 19, in sales_summary
    rows, by_region = load_rows(month), defaultdict(int)
                      ^^^^^^^^^^^^^^^^
  File "/work/sales-mcp/bad_relpath.py", line 11, in load_rows
    ws = load_workbook(DATA, read_only=True).active
         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/work/sales-mcp/.venv/lib/python3.12/site-packages/openpyxl/reader/excel.py", line 346, in load_workbook
    reader = ExcelReader(filename, read_only, keep_vba,
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/work/sales-mcp/.venv/lib/python3.12/site-packages/openpyxl/reader/excel.py", line 123, in __init__
    self.archive = _validate_archive(fn)
                   ^^^^^^^^^^^^^^^^^^^^^
  File "/work/sales-mcp/.venv/lib/python3.12/site-packages/openpyxl/reader/excel.py", line 95, in _validate_archive
    archive = ZipFile(filename, 'r')
              ^^^^^^^^^^^^^^^^^^^^^^
  File "/root/.local/share/uv/python/cpython-3.12.15-linux-aarch64-gnu/lib/python3.12/zipfile/__init__.py", line 1378, in __init__
    self.fp = io.open(file, filemode)
              ^^^^^^^^^^^^^^^^^^^^^^^
FileNotFoundError: [Errno 2] No such file or directory: 'sales.xlsx'

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/work/sales-mcp/.venv/lib/python3.12/site-packages/mcp/server/mcpserver/server.py", line 442, in _handle_call_tool
    return await self.call_tool(params.name, params.arguments or {}, context)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/work/sales-mcp/.venv/lib/python3.12/site-packages/mcp/server/mcpserver/server.py", line 548, in call_tool
    return await self._tool_manager.call_tool(name, arguments, context, convert_result=True)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/work/sales-mcp/.venv/lib/python3.12/site-packages/mcp/server/mcpserver/tools/tool_manager.py", line 87, in call_tool
    return await tool.run(arguments, context, convert_result=convert_result)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/work/sales-mcp/.venv/lib/python3.12/site-packages/mcp/server/mcpserver/tools/base.py", line 213, in run
    raise UnexpectedToolError(f"Error executing tool {self.name}") from exc
mcp.server.mcpserver.exceptions.UnexpectedToolError: Error executing tool sales_summary
{
  "content": [
    {
      "type": "text",
      "text": "Error executing tool sales_summary"
    }
  ],
  "isError": true
}
{"error":{"code":"tool_is_error","message":"Tool 'sales_summary' returned isError:true."}}

```

## r1-s11 · exit 0 · 2.4s

```
$ export PATH="$HOME/.local/bin:$PATH"
echo "터미널 PATH: $PATH"
echo "터미널에서 uv: $(command -v uv)"
echo '=== GUI 앱처럼 짧은 PATH(/usr/local/bin:/usr/bin:/bin)에서 uv 찾기 ==='
env PATH=/usr/local/bin:/usr/bin:/bin sh -c 'command -v uv; echo "command -v uv exit: $?"'
cd /tmp
echo '=== [실패 재현 3] command: "uv" (이름만) ==='
env PATH=/usr/local/bin:/usr/bin:/bin npx -y @modelcontextprotocol/inspector --cli --config /work/inspector-lab.json --server sales-bare-uv --method tools/list > /work/logs/bare_uv.txt 2>&1
echo "exit code: $?"
cat /work/logs/bare_uv.txt
echo '=== [해결] command: uv 절대경로 ==='
env PATH=/usr/local/bin:/usr/bin:/bin npx -y @modelcontextprotocol/inspector --cli --config /work/inspector-lab.json --server sales --method tools/list > /work/logs/abs_uv.txt 2>&1
echo "exit code: $?"
cat /work/logs/abs_uv.txt
터미널 PATH: /root/.local/bin:/root/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
터미널에서 uv: /root/.local/bin/uv
=== GUI 앱처럼 짧은 PATH(/usr/local/bin:/usr/bin:/bin)에서 uv 찾기 ===
command -v uv exit: 127
=== [실패 재현 3] command: "uv" (이름만) ===
exit code: 1

[mcp-inspector] The OS keychain is not available, so secrets will be kept in: Memory (this session only) — fell back from the OS keychain.
[mcp-inspector] Keychain error: Couldn't access platform storage: AccessDenied

Caused by:
    AccessDenied
[mcp-inspector] Set MCP_INSPECTOR_SECRET_STORE=keyring|file|memory to choose explicitly.
[mcp-inspector] Secrets are not written anywhere and are lost on exit.
[mcp-inspector] How the secret store is chosen, and how to secure it: https://github.com/modelcontextprotocol/inspector/blob/main/docs/secret-storage.md
{"error":{"code":"error","message":"spawn uv ENOENT"}}
=== [해결] command: uv 절대경로 ===
exit code: 0

[mcp-inspector] The OS keychain is not available, so secrets will be kept in: Memory (this session only) — fell back from the OS keychain.
[mcp-inspector] Keychain error: Couldn't access platform storage: AccessDenied

Caused by:
    AccessDenied
[mcp-inspector] Set MCP_INSPECTOR_SECRET_STORE=keyring|file|memory to choose explicitly.
[mcp-inspector] Secrets are not written anywhere and are lost on exit.
[mcp-inspector] How the secret store is chosen, and how to secure it: https://github.com/modelcontextprotocol/inspector/blob/main/docs/secret-storage.md
sales MCP server starting
{
  "tools": [
    {
      "name": "sales_summary",
      "description": "월별 매출 요약(총매출, 주문 수, 지역별 매출). month 는 'YYYY-MM' 형식 (예: 2026-09).",
      "inputSchema": {
        "type": "object",
        "properties": {
          "month": {
            "title": "Month",
            "type": "string"
          }
        },
        "required": [
          "month"
        ],
        "title": "sales_summaryArguments"
      }
    },
    {
      "name": "top_products",
      "description": "해당 월 매출 상위 n개 상품과 판매 수량, 매출.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "month": {
            "title": "Month",
            "type": "string"
          },
          "n": {
            "default": 3,
            "title": "N",
            "type": "integer"
          }
        },
        "required": [
          "month"
        ],
        "title": "top_productsArguments"
      },
      "outputSchema": {
        "type": "object",
        "properties": {
          "result": {
            "items": {
              "additionalProperties": true,
              "type": "object"
            },
            "title": "Result",
            "type": "array"
          }
        },
        "required": [
          "result"
        ],
        "title": "top_productsOutput"
      }
    }
  ]
}

```

## r1-s12 · exit 0 · 18.3s

```
$ set -euo pipefail
export PATH="$HOME/.local/bin:$PATH"
npm install -g @anthropic-ai/claude-code
echo "claude: $(claude --version)"
cd /work/sales-mcp
claude mcp add --scope local sales -- "$(command -v uv)" run --directory /work/sales-mcp server.py
claude mcp add --scope project sales-team -- "$(command -v uv)" run --directory /work/sales-mcp server.py
echo '=== .mcp.json (project scope, 팀과 git 으로 공유) ==='
cat .mcp.json
echo '=== claude mcp list ==='
claude mcp list
echo '=== claude mcp get sales ==='
claude mcp get sales

added 2 packages in 15s
claude: 2.1.292 (Claude Code)
Added stdio MCP server sales with command: /root/.local/bin/uv run --directory /work/sales-mcp server.py to local config
File modified: /root/.claude.json [project: /work/sales-mcp]
Added stdio MCP server sales-team with command: /root/.local/bin/uv run --directory /work/sales-mcp server.py to project config
File modified: /work/sales-mcp/.mcp.json
=== .mcp.json (project scope, 팀과 git 으로 공유) ===
{
  "mcpServers": {
    "sales-team": {
      "type": "stdio",
      "command": "/root/.local/bin/uv",
      "args": [
        "run",
        "--directory",
        "/work/sales-mcp",
        "server.py"
      ],
      "env": {}
    }
  }
}=== claude mcp list ===
Checking MCP server health…

sales-team: /root/.local/bin/uv run --directory /work/sales-mcp server.py - ⏸ Pending approval (run `claude` to approve)
sales: /root/.local/bin/uv run --directory /work/sales-mcp server.py - ✔ Connected
=== claude mcp get sales ===
sales:
  Scope: Local config (private to you in this project)
  Status: ✔ Connected
  Type: stdio
  Command: /root/.local/bin/uv
  Args: run --directory /work/sales-mcp server.py
  Environment:

To remove this server, run: claude mcp remove sales -s local

```

## r1-s13 · exit 0 · 13.2s

```
$ set -euo pipefail
export PATH="$HOME/.local/bin:$PATH"
cat > /work/scripts/count_tokens.py <<'EOF'
import json, sys
import tiktoken

text = open(sys.argv[1], encoding="utf-8").read()
data = json.loads(text[text.index("{"):])
tools = data["tools"]
enc = tiktoken.get_encoding("o200k_base")
print("* tiktoken(o200k_base) 근사치 — Claude 토크나이저와 다름")
all_json = json.dumps(tools, ensure_ascii=False, separators=(",", ":"))
print(f"tools: {len(tools)}")
print(f"all tools  bytes={len(all_json.encode('utf-8'))} chars={len(all_json)} tokens~={len(enc.encode(all_json))}")
for t in tools:
    s = json.dumps(t, ensure_ascii=False, separators=(",", ":"))
    d = t.get("description", "")
    print(f"- {t['name']}: bytes={len(s.encode('utf-8'))} tokens~={len(enc.encode(s))} (description tokens~={len(enc.encode(d))})")
EOF
cd /work/sales-mcp
uv run --with tiktoken python /work/scripts/count_tokens.py /work/logs/tools_list.json
Downloading tiktoken (1.1MiB)
 Downloaded tiktoken
Installed 7 packages in 36ms
* tiktoken(o200k_base) 근사치 — Claude 토크나이저와 다름
tools: 2
all tools  bytes=779 chars=703 tokens~=194
- sales_summary: bytes=289 tokens~=76 (description tokens~=36)
- top_products: bytes=487 tokens~=116 (description tokens~=18)

```

## r1-s14 · exit 0 · 0.1s

```
$ export PATH="$HOME/.local/bin:$PATH"
cd /work
DANGEROUSLY_OMIT_AUTH=true MCP_AUTO_OPEN_ENABLED=false npx -y @modelcontextprotocol/inspector
26cfb5e4c6760ececbfefe470b15e09eea6689f2ee4e7a750a027079d11f4028

```

## w2

```
URL: https://py.sdk.modelcontextprotocol.io/migration/
Skip to content
MCP Python SDK
Migration Guide: v1 to v2
en - English
de - Deutsch
es - español
fr - français
hi - हिन्दी
ja - 日本語
ko - 한국어
pt - português (Brasil)
ru - русский язык
tr - Türkçe
uk - українська мова
zh - 简体中文
zh-hant - 繁體中文
Search
 modelcontextprotocol/python-sdk
v2.3.0
24.5k
4k
MCP Python SDK
What's new in v2
GET STARTED
Installation
First steps
Connect to a real host
Testing
SERVERS
Tools
Structured Output
Resources
URI templates
Prompts
Completions
Images, audio & icons
Handling errors
INSIDE YOUR HANDLER
The Context
Dependencies
Lifespan
Elicitation
Multi-round-trip requests
Sampling and roots
Progress
Cancellation
Logging
Subscriptions
RUNNING YOUR SERVER
Add to an existing app
Deploy & scale
Authorization
OpenTelemetry
Serving legacy clients
CLIENTS
Callbacks
Transports
OAuth
Identity assertion
Multiple servers
Subscriptions
Caching
Protocol versions
Deprecated features
ADVANCED
The low-level Server
Pagination
Middleware
Header parameters
Extensions
MCP Apps
Troubleshooting
Translations
Migration Guide
API REFERENCE
mcp
mcp_types
On this page
Find your changes
Changes almost every project hits
Find your area
Suggested migration order
Packaging, dependencies, and CLI
Dependency floors raised and new required dependencies
httpx and httpx-sse replaced by httpx2
mcp dev and mcp install pin the spawned environment to your SDK version
Types and wire format
mcp.types moved to the mcp-types package
Removed type aliases and classes
Field names changed from camelCase to snake_case
Extra fields on MCP types are no longer preserved
Resource URI type changed from AnyUrl to str
Replace RootModel by union types with TypeAdapter validation
RequestParams.Meta replaced with RequestParamsMeta TypedDict
SUPPORTED_PROTOCOL_VERSIONS deprecated; LATEST_PROTOCOL_VERSION changed meaning
McpError renamed to MCPError
JSONRPCError.id is now RequestId | None
MCPServer (formerly FastMCP)
FastMCP renamed to MCPServer
What is unchanged on MCPServer
Default server name changed from FastMCP to mcp-server
MCPServer constructor: title, description, and version added to the positional parameters
Unversioned servers report an empty version
mount_path parameter removed from MCPServer
Transport-specific parameters moved from MCPServer constructor to run()/app methods
MCP_* environment variables and .env files are no longer read
Streamable HTTP request bodies are limited to 4 MiB
Streamable HTTP: lifespan now entered once at manager startup
Streamable HTTP: session manager, EventStore, and stateless mode unchanged
MCPServer.get_context() removed
Sync handler functions now run on a worker thread
MCPServer.call_tool() returns CallToolResult
MCPServer.get_prompt() and read_resource() may return InputRequiredResult
MCPServer.call_tool(), read_resource(), get_prompt() now accept a context parameter
Resolver-routed requests require the client capability on every protocol version
MCPError raised from an @mcp.tool() handler now surfaces as a JSON-RPC error
Resource not found returns -32602 and resource lookups raise typed exceptions (SEP-2164)
Resource classes reject unknown keyword arguments
FileResource.is_binary replaced by encoding
Resource templates: matching behavior changes
MCPServer's Context logging: message renamed to data, extra removed
Context.client_id removed
ProgressContext and progress() context manager removed
Context.elicit() schema gate validates the rendered schema
isinstance() checks against ElicitationResult raise TypeError
Registering lowlevel handlers from MCPServer
Lowlevel Server
Lowlevel Server: what did not change
Lowlevel Server: decorator-based handlers replaced with constructor on_* params
Lowlevel Server: automatic return value wrapping removed
Lowlevel Server: tool handler exceptions no longer become CallToolResult(is_error=True)
Lowlevel Server: constructor parameters are now keyword-only
Lowlevel Server: type parameter reduced from 2 to 1
Lowlevel Server: request_handlers and notification_handlers attributes removed
Lowlevel Server: subscribe capability now correctly reported
Lowlevel Server: private _handle_* dispatch methods removed
Lowlevel Server.run(raise_exceptions=True): transport errors no longer re-raised
Cancelled requests are no longer answered
Server.run() no longer takes a stateless flag
Lowlevel Server: request_context property removed
RequestContext type parameters simplified
ServerSession is now a thin proxy (no longer a BaseSession)
ServerSession.elicit() and elicit_form() take requested_schema, not requestedSchema
Clients
Client defaults to mode='auto'
ClientSession.get_server_capabilities() replaced by era-neutral accessors
cursor parameter removed from ClientSession list methods
args parameter removed from ClientSessionGroup.call_tool()
Timeouts take float seconds instead of timedelta
Client request timeouts now raise -32001 (REQUEST_TIMEOUT) instead of 408
ClientSession now runs on JSONRPCDispatcher; BaseSession removed
Experimental Tasks support removed
Transports
streamablehttp_client removed
get_session_id callback removed from streamable_http_client
StreamableHTTPTransport parameters removed
StreamableHTTPTransport.protocol_version attribute removed
Streamable HTTP: non-2xx responses now surface as per-request JSON-RPC errors
terminate_windows_process removed
stdio_client shutdown reworked: a gracefully-exited server's children are left alive on POSIX
stdio_server keeps the protocol streams on private descriptors
WebSocket transport removed
OAuth and server auth
Unchanged auth surfaces
RFC7523OAuthClientProvider and JWTParameters removed
OAuth metadata URLs no longer gain a trailing slash
OAuth callback_handler returns AuthorizationCodeResult
scopes= renamed to scope= on the client-credentials providers
client_secret_post token requests now include client_id
timeout parameter removed from OAuthClientProvider
Client rejects authorization server metadata with a mismatched issuer
OAuth client requests offline_access and adds prompt=consent when the authorization server supports it (SEP-2207)
OAuth client credentials are bound to their authorization server (SEP-2352)
Step-up authorization unions previously requested scopes (SEP-2350)
OAuth Dynamic Client Registration sends application_type (SEP-837)
OAuthClientInformationFull no longer subclasses OAuthClientMetadata, and parses server-substituted metadata
Stricter client authentication at /token and /revoke
Stricter protocol validation and wire behavior
Server handler results are validated against the protocol schema
Client validates inbound traffic against the protocol schema
Unknown request methods now return -32601 (Method not found)
OpenTelemetry is on by default
Testing utilities
create_connected_server_and_client_session removed
Deprecations
Client resource-subscription methods deprecated (SEP-2575)
Roots, Sampling, and Logging methods deprecated (SEP-2577)
Client-to-server progress deprecated (2026-07-28)
Notes for 2026-era connections
Server-initiated sampling, elicitation, and roots raise NoBackChannelError
Log messages are delivered only to requests that opt in
Change notifications travel only on subscriptions/listen streams
Need Help?
Migration Guide: v1 to v2

This guide covers the breaking changes introduced in v2 of the MCP Python SDK and how to update your code.

Version 2 of the MCP Python SDK introduces several breaking changes to improve the API, align with the MCP specification, and provide better type safety.

Not ready to migrate yet?

The v1.x maintenance line keeps receiving critical bug fixes and security patches, and its documentation is at /v1/. If your package depends on mcp, keep a <2 upper bound until you've migrated.

Find your changes

Every section heading below names the API it affects, so searching this page for the symbol your code uses is the fastest route to the change that broke it. The guide lists changes only: an SDK API not mentioned here behaves as it did in v1, and the "what did not change" summaries — MCPServer, lowlevel Server, and auth — spell out the surfaces most migrators stop to check.

Changes almost every project hits
Change	First symptom	Section
FastMCP renamed to MCPServer	ModuleNotFoundError: No module named 'mcp.server.fastmcp' (newer 2.x releases follow it with a pointer to this guide)	FastMCP renamed
Fields renamed from camelCase to snake_case	AttributeError: 'Tool' object has no attribute 'inputSchema'	snake_case fields
mcp.types names removed	ImportError: cannot import name 'Content' from 'mcp.types'	Removed types
McpError renamed to MCPError	ImportError: cannot import name 'McpError' from 'mcp'	McpError renamed
Resource URIs are str, not AnyUrl	AttributeError: 'str' object has no attribute 'host'	URI type
Message unions (ServerNotification, JSONRPCMessage, ...) are plain unions, not RootModel	AttributeError: 'LoggingMessageNotification' object has no attribute 'root'	RootModel → unions
streamablehttp_client removed	ImportError: cannot import name 'streamablehttp_client'	streamablehttp_client
httpx and httpx-sse replaced by httpx2	ModuleNotFoundError: No module named 'httpx', or TypeError: Invalid "auth" argument from httpx.AsyncClient(auth=provider)	httpx2 swap
Client defaults to mode='auto'	servers log an unexpected server/discover request	mode='auto'
Transport parameters moved off the MCPServer constructor	TypeError: MCPServer.__init__() got an unexpected keyword argument 'port'	constructor parameters
Sync handlers run on a worker thread	asyncio.get_running_loop() in a def handler raises RuntimeError	worker threads
Lowlevel decorators replaced with on_* constructor params	AttributeError: 'Server' object has no attribute 'list_tools'	on_* handlers
Lowlevel return value wrapping removed	bare list or dict returns fail result validation instead of being wrapped	wrapping removed
Lowlevel tool exceptions no longer become isError: true results	clients raise a JSON-RPC error instead of seeing the error text	tool exceptions
Roots, Sampling, and Logging deprecated (SEP-2577)	MCPDeprecationWarning at call sites	SEP-2577
Find your area
If you...	Read
pin dependencies or use the mcp CLI	Packaging, dependencies, and CLI
import mcp.types or touch protocol types (everyone does)	Types and wire format
run FastMCP/MCPServer servers	MCPServer (formerly FastMCP)
use the lowlevel Server	Lowlevel Server, plus Timeouts take float seconds and Experimental Tasks support removed under Clients
write client code with Client or ClientSession	Clients, plus streamablehttp_client removed under Transports
use stdio or streamable HTTP directly, or maintain a custom transport	Transports
maintain OAuth client auth or a protected server	OAuth and server auth
relied on lenient handling of off-schema traffic, or assert on exact wire bytes	Stricter protocol validation and wire behavior
test against in-memory server/client pairs	Testing utilities
use roots, sampling, logging, or client-to-server progress	Deprecations
operate servers that 2026-era clients will also connect to	Notes for 2026-era connections
Suggested migration order
Update your dependency pins and CLI usage: Packaging, dependencies, and CLI.
Apply the mechanical renames and import moves: Types and wire format.
Port your server surface: MCPServer (formerly FastMCP) or Lowlevel Server.
Port your client code: Clients.
Update transport setup and auth: Transports and OAuth and server auth.
Run your tests and check anything that now errors against Stricter protocol validation and wire behavior and Testing utilities.
Address deprecation warnings: Deprecations.
Packaging, dependencies, and CLI
Dependency floors raised and new required dependencies

v2 raises the minimum versions of several shared dependencies and adds new required ones. A project that pins any of these below the new floor fails dependency resolution before anything installs (uv reports "No solution found when resolving dependencies"; pip fails similarly).

Dependency	v1.28.1	v2	Change
anyio	>=4.5	>=4.9 (Python <3.14) / >=4.10 (Python >=3.14)	floor raised
pydantic	>=2.11,<3 (Python <3.14)	>=2.12	floor raised on Python <3.14; <3 cap dropped
sse-starlette	>=1.6.1	>=3.0.0	floor raised across two majors
typing-extensions	>=4.9.0	>=4.13.0	floor raised
pywin32 (Windows)	>=310 (Python <3.14)	>=311	floor raised on Python <3.14
opentelemetry-api	not a dependency	>=1.28.0	new required dependency
mcp-types	not a dependency	==<exact mcp version>	new, exact-pinned
httpx	>=0.27.1,<1.0.0	removed	see httpx and httpx-sse replaced by httpx2
httpx-sse	>=0.4	removed	see httpx and httpx-sse replaced by httpx2
httpx2	not a dependency	>=2.5.0	new required dependency
ws extra	websockets>=15.0.1	removed	see WebSocket transport removed

Before (v1):

dependencies = [
    "mcp==1.28.1",
    "sse-starlette>=2,<3",  # own SSE endpoints, pinned to the 2.x API
]


After (v2):

dependencies = [
    "mcp>=2,<3",
    "sse-starlette>=3",  # absorb sse-starlette's own 2.x -> 3.x changes
]


Relax or bump any conflicting pins when upgrading. sse-starlette jumps two majors, so a project that imports sse_starlette itself must also work through that library's own breaking changes to co-install with mcp v2. opentelemetry-api is a new hard dependency because OpenTelemetry trace propagation now ships enabled; see OpenTelemetry is on by default. mcp-types is exact-pinned to the SDK version; nothing in a v1 tree can conflict with it, but do not pin mcp-types independently of mcp.

httpx and httpx-sse replaced by httpx2

The SDK now depends on httpx2 instead of httpx and httpx-sse. httpx2 is the next-generation HTTP client (a fork of httpx) with server-sent events support built in, so the separate httpx-sse dependency is gone.

The swap changes types, not parameter lists: streamable_http_client and sse_client keep their keyword arguments (covered, with the removed streamablehttp_client alias and the get_session_id callback, under Transports), and only the objects they take become httpx2 types — the pre-built http_client you hand streamable_http_client, sse_client's auth= (an httpx2.Auth, the base class OAuthClientProvider now uses), and the client a custom httpx_client_factory returns. Import from httpx2 when building any of them:

Before (v1):

import httpx

http_client = httpx.AsyncClient(timeout=httpx.Timeout(30, read=300))


After (v2):

import httpx2

http_client = httpx2.AsyncClient(timeout=httpx2.Timeout(30, read=300))


httpx2 is API-compatible with httpx, so usually only the import name changes. To consume SSE directly, use httpx2.EventSource (or AsyncClient.sse()) instead of the httpx-sse helpers.

mcp no longer installs httpx at all. If your own code imports httpx and relied on mcp v1 to pull it in, that import now fails with ModuleNotFoundError: No module named 'httpx' — a traceback that never mentions mcp. Either add httpx to your own dependencies (the two packages install side by side; only objects handed to the SDK via http_client= or auth= have to be httpx2 types) or port those calls to httpx2, whose Client and AsyncClient are drop-in replacements.

Exception handlers need the same rename: the SDK now raises httpx2 exceptions (httpx2.ConnectError, httpx2.HTTPStatusError, and so on), and this failure mode is silent. If httpx is still installed — your own code or another package depends on it — an old except httpx.ConnectError: block keeps importing fine and simply never matches again. Audit except httpx. clauses and isinstance checks along with the imports, and switch test fixtures in the same change: pytest.raises(httpx.ConnectError), an httpx.MockTransport, or a test-only httpx.Auth subclass all target the wrong types once the code under test moves to httpx2. The same identity split applies to objects: httpx and httpx2 types are not interchangeable at runtime, so an httpx.AsyncClient passed as http_client degrades in subtle ways (server-initiated messages stop arriving) instead of raising immediately.

Retry and error-classification logic keyed to HTTP status codes needs a look too: through the SDK's client, timeouts and non-2xx responses surface as MCPError with JSON-RPC codes, not 408s or httpx.HTTPStatusError — see the client request timeouts section (REQUEST_TIMEOUT, -32001) under Clients and Streamable HTTP: non-2xx responses now surface as per-request JSON-RPC errors.

The SDK's own auth providers made the same move: OAuthClientProvider, ClientCredentialsOAuthProvider, PrivateKeyJWTOAuthProvider, and IdentityAssertionOAuthProvider now subclass httpx2.Auth (v1: httpx.Auth), so the client you attach one to must be an httpx2.AsyncClient. Unlike the silent http_client degradation, this direction fails loudly at construction: httpx.AsyncClient(auth=provider) raises TypeError: Invalid "auth" argument. See OAuth clients for the httpx2.AsyncClient(auth=...) wiring.

The client also identifies itself differently: the default User-Agent is now python-httpx2/<version>, and log lines come from the httpx2 and httpcore2.* loggers, so a logging.getLogger("httpx") or logging.getLogger("httpcore") suppression no longer matches anything — target logging.getLogger("httpx2") and logging.getLogger("httpcore2") instead. Telemetry integrations keyed to the httpx module (such as OpenTelemetry's httpx instrumentation) stop seeing the SDK's traffic as well.

TLS verification also changes: httpx validated certificates against the bundled certifi CA list, while httpx2 validates against the operating system trust store via truststore. If your environment has no usable system CA store (some minimal containers), or you relied on certifi's bundle specifically, point the standard SSL_CERT_FILE or SSL_CERT_DIR environment variable at a CA bundle (httpx2 honors these before falling back to the system store), or pass an explicit verify=ssl_context to your httpx2.AsyncClient. Passing a CA bundle path as verify="ca.pem" or using the cert= parameter is deprecated in httpx2; build an ssl.SSLContext and configure it instead.

mcp dev and mcp install pin the spawned environment to your SDK version

Both commands run your server through a fresh uv run --with ... environment. In v1 the mcp requirement in that command was unpinned, so the spawned environment resolved to the newest stable release rather than the version you had installed; while v2 was in pre-release, mcp dev server.py built a v1 environment that could not import a v2 server. Both commands now pin the requirement to the version you are running (mcp==<installed version>). Source builds and other unpublished versions, which have nothing on PyPI to pin to, keep the unpinned form.

Types and wire format
mcp.types moved to the mcp-types package

The protocol wire types now live in a standalone distribution, mcp-types (import package mcp_types). Its only runtime dependencies are pydantic and typing-extensions, so code that just needs to (de)serialize MCP traffic can install it without the full SDK. Its API reference is at mcp_types.

If your project depends on mcp, nothing changes for you. import mcp.types, from mcp.types import ..., from mcp import types, and import mcp followed by mcp.types.Tool all keep working: mcp.types is a permanent alias that mirrors mcp_types exactly (every name is the same object), and mcp.types.version mirrors mcp_types.version the same way. Keep importing through mcp — the package you actually depend on — rather than writing import mcp_types, which would reach past your declared dependency into a transitive one. The old mcp.shared.version module was removed; import the version registry from mcp.types.version instead. The top-level from mcp import Tool re-exports are unchanged too.

Import mcp_types directly only in a project that depends on mcp-types without the SDK. That is the point of the split: tooling and lightweight clients can depend on the protocol schema without pulling in httpx2, starlette, uvicorn, and the rest of the server/transport stack.

Names that no longer exist (listed under Removed type aliases and classes) fail on import or attribute access with an ordinary ImportError / AttributeError; the table below names each replacement.

The supported import surface is the package plus its jsonrpc, methods, and version submodules, and each has both spellings: mcp.types / mcp_types, mcp.types.jsonrpc / mcp_types.jsonrpc, mcp.types.methods / mcp_types.methods, and mcp.types.version / mcp_types.version (each mcp.types module mirrors its mcp_types counterpart, name for name, the same objects). Underscore-prefixed submodules (mcp_types._types, and the generated per-protocol-version packages mcp_types._v2025_11_25 / mcp_types._v2026_07_28) are internal validators with unstable class names; don't import from them, under either spelling.

Before (v1):

from mcp.types import Tool, Resource
from mcp.shared.version import LATEST_PROTOCOL_VERSION


After (v2), depending on mcp:

from mcp.types import Tool, Resource  # unchanged
from mcp.types.version import LATEST_PROTOCOL_VERSION


After (v2), depending only on mcp-types (no SDK):

from mcp_types import Tool, Resource
from mcp_types.version import LATEST_PROTOCOL_VERSION

Removed type aliases and classes

The following type aliases and classes have been removed from the protocol types (mcp.types / mcp_types):

Removed	Replacement
Content	ContentBlock
ResourceReference	ResourceTemplateReference
Cursor	Use str directly
MethodT	Internal TypeVar, not intended for public use
RequestParamsT	Internal TypeVar, not intended for public use
NotificationParamsT	Internal TypeVar, not intended for public use
AnyFunction	Use Callable[..., Any] directly
ClientRequestType, ClientNotificationType, ClientResultType, ServerRequestType, ServerNotificationType, ServerResultType	The union is now the bare name: ClientRequest, ClientNotification, ClientResult, ServerRequest, ServerNotification, ServerResult
TaskExecutionMode, TASK_FORBIDDEN, TASK_OPTIONAL, TASK_REQUIRED, TASK_STATUS_*	Use string literals; TaskStatus remains as the literal-union type

Before (v1):

from mcp.types import Content, ResourceReference, Cursor


After (v2):

from mcp.types import ContentBlock, ResourceTemplateReference
# Use `str` instead of `Cursor` for pagination cursors

Field names changed from camelCase to snake_case

All Pydantic model fields in the protocol types now use snake_case names for Python attribute access. The JSON wire format is unchanged — traffic the SDK sends still uses camelCase via Pydantic aliases, but your own model_dump() calls now need by_alias=True to produce it.

Before (v1):

result = await session.call_tool("my_tool", {"x": 1})
if result.isError:
    ...

tools = await session.list_tools()
cursor = tools.nextCursor
schema = tools.tools[0].inputSchema


After (v2):

result = await session.call_tool("my_tool", {"x": 1})
if result.is_error:
    ...

tools = await session.list_tools()
cursor = tools.next_cursor
schema = tools.tools[0].input_schema


Common renames:

v1 (camelCase)	v2 (snake_case)
inputSchema	input_schema
outputSchema	output_schema
isError	is_error
nextCursor	next_cursor
mimeType	mime_type
structuredContent	structured_content
serverInfo	server_info
protocolVersion	protocol_version
uriTemplate	uri_template
listChanged	list_changed
progressToken	progress_token

The models accept both spellings at construction time, so the old camelCase names still work as constructor kwargs (e.g., Tool(inputSchema={...}) is accepted), but attribute access must use snake_case (tool.input_schema).

If you serialize models yourself, pass by_alias=True. In v1, model_dump() produced wire-format camelCase keys because the fields themselves were camelCase. In v2 the same call emits snake_case keys (input_schema, not inputSchema), which peers and other MCP implementations will not recognize. No error is raised; the output is silently in the wrong shape.

tool.model_dump()                                # {"name": ..., "input_schema": ...}
tool.model_dump(by_alias=True, mode="json")      # {"name": ..., "inputSchema": ...}  (wire format)


Parsing is unaffected: model_validate() accepts both camelCase wire JSON and snake_case dumps.

Extra fields on MCP types are no longer preserved

In v1, MCP protocol types were configured with extra="allow": unknown fields passed to a constructor or received from a peer were kept on the model and re-serialized on output.

In v2, MCP types silently ignore extra fields. Unknown constructor keyword arguments and unknown keys in wire data are dropped during validation — no error is raised, and the values do not round-trip:

from mcp.types import CallToolRequestParams

params = CallToolRequestParams(
    name="my_tool",
    arguments={},
    unknown_field="value",  # silently ignored, not stored
)
"unknown_field" in params.model_dump()  # False

# _meta remains the supported place for custom data, per the MCP spec
params = CallToolRequestParams(
    name="my_tool",
    arguments={},
    _meta={"my_custom_key": "value", "another": 123},  # OK, preserved
)


If you relied on extra fields round-tripping through MCP types, move that data into _meta.

Resource URI type changed from AnyUrl to str

The uri field on resource-related types now uses str instead of Pydantic's AnyUrl. This aligns with the MCP specification schema which defines URIs as plain strings (uri: string) without strict URL validation. This change allows relative paths like users/me that were previously rejected.

Before (v1):

from pydantic import AnyUrl
from mcp.types import Resource

# uri was typed as AnyUrl; relative paths were rejected
resource = Resource(name="test", uri=AnyUrl("users/me"))  # Would fail validation


After (v2):

from mcp.types import Resource

# Plain strings accepted
resource = Resource(name="test", uri="users/me")  # Works
resource = Resource(name="test", uri="custom://scheme")  # Works
resource = Resource(name="test", uri="https://example.com")  # Works


If your code passes AnyUrl objects to URI fields, convert them to strings:

# If you have an AnyUrl from elsewhere
uri = str(my_any_url)  # Convert to string


Affected types:

Resource.uri (and subclass ResourceLink)
ReadResourceRequestParams.uri
ResourceContents.uri (and subclasses TextResourceContents, BlobResourceContents)
SubscribeRequestParams.uri
UnsubscribeRequestParams.uri
ResourceUpdatedNotificationParams.uri

The Client and ClientSession methods read_resource(), subscribe_resource(), and unsubscribe_resource() now only accept str for the uri parameter. If you were passing AnyUrl objects, convert them to strings:

# Before (v1)
from pydantic import AnyUrl

await client.read_resource(AnyUrl("test://resource"))

# After (v2)
await client.read_resource("test://resource")
# Or if you have an AnyUrl from elsewhere:
await client.read_resource(str(my_any_url))


URI values you read back are also plain strings now. In v1, fields like Resource.uri and ResourceContents.uri were AnyUrl objects, so attribute access such as uri.scheme or uri.host worked; in v2 that code raises AttributeError. Use urllib.parse if you need to parse them. Note that v1 also normalized URIs during validation (for example https://example.com became https://example.com/), while v2 preserves the string exactly as given, so URIs sent on the wire may differ byte-for-byte from what v1 sent.

Replace RootModel by union types with TypeAdapter validation

The following union types are no longer RootModel subclasses:

ClientRequest
ServerRequest
ClientNotification
ServerNotification
ClientResult
ServerResult
JSONRPCMessage

This means you can no longer access .root on these types or use model_validate() directly on them. Instead, use the provided TypeAdapter instances for validation.

Before (v1):

from mcp.types import ClientRequest, ServerNotification

# Using RootModel.model_validate()
request = ClientRequest.model_validate(data)
actual_request = request.root  # Accessing the wrapped value

notification = ServerNotification.model_validate(data)
actual_notification = notification.root


After (v2):

from mcp.types import client_request_adapter, server_notification_adapter

# Using TypeAdapter.validate_python()
request = client_request_adapter.validate_python(data)
# No .root access needed - request is the actual type

notification = server_notification_adapter.validate_python(data)
# No .root access needed - notification is the actual type


The same applies when constructing values — the wrapper call is no longer needed:

Before (v1):

await session.send_notification(ClientNotification(InitializedNotification()))
await session.send_request(ClientRequest(PingRequest()), EmptyResult)


After (v2):

await session.send_notification(InitializedNotification())
await session.send_request(PingRequest(), EmptyResult)

# Params are constructed as before; only the outer wrapper is gone
await session.send_notification(
    CancelledNotification(params=CancelledNotificationParams(request_id=request_id, reason="timeout"))
)


Available adapters:

Union Type	Adapter
ClientRequest	client_request_adapter
ServerRequest	server_request_adapter
ClientNotification	client_notification_adapter
ServerNotification	server_notification_adapter
ClientResult	client_result_adapter
ServerResult	server_result_adapter
JSONRPCMessage	jsonrpc_message_adapter

All adapters are exported from mcp.types.

These are ordinary X | Y unions of the concrete pydantic classes, so isinstance(msg, ServerNotification), isinstance(msg, LoggingMessageNotification), and match/case on the member classes keep working (unlike ElicitationResult, which became a TypeAliasType — see isinstance() checks against ElicitationResult raise TypeError).

Values the SDK hands you are the member instances themselves, so delete .root accesses. A message_handler, for example, now receives the notification directly (v1 code fails with AttributeError: 'LoggingMessageNotification' object has no attribute 'root'):

# Before (v1)
if isinstance(message, ServerNotification):
    if isinstance(message.root, LoggingMessageNotification):
        print(message.root.params.data)

# After (v2)
if isins
```

## w3

```
URL: https://pypi.org/project/mcp/
메인 컨텐츠로 가기
PyPI 검색
검색
도움말 문서 로그인 가입하기
mcp 2.3.0

Model Context Protocol SDK

pip install mcp
PIP 사용법 복사

 Description
 Release files
 릴리즈 이력
MCP Python SDK

Python implementation of the Model Context Protocol (MCP)

     

Documentation

The documentation lives at https://py.sdk.modelcontextprotocol.io/.

It has a Get started guide, What's new in v2, the API reference, and the migration guide.

What is MCP?

The Model Context Protocol lets you build servers that expose data and functionality to LLM applications in a secure, standardized way. Think of it like a web API, but designed for LLM interactions. With this SDK you can:

Build MCP servers that expose tools, resources, and prompts to any MCP host
Build MCP clients that connect to any MCP server
Speak every standard transport: stdio, Streamable HTTP, and SSE
Requirements

Python 3.10+.

Installation
uv add "mcp[cli]"      # or: pip install "mcp[cli]"


The cli extra adds the mcp command-line tool (mcp dev, mcp run, mcp install) on top of the SDK; install plain mcp if you don't need it. For one-off commands, uv run --with "mcp[cli]" mcp ... works without a project.

A server in 15 lines

Create a server.py:

from mcp.server import MCPServer

mcp = MCPServer("Demo")


@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b


@mcp.resource("greeting://{name}")
def greeting(name: str) -> str:
    """Greet someone by name."""
    return f"Hello, {name}!"


Full example: docs_src/index/tutorial001.py

That's a complete MCP server: one tool, one templated resource. Open it in the MCP Inspector:

uv run mcp dev server.py


Call add with a=1, b=2 and you get 3 back.

Notice what you did not write: no JSON Schema (a: int, b: int is the schema), no request parsing, no validation code, no protocol handling. Two type-hinted Python functions and a docstring.

Get started takes it from here.

A client in 10 lines

The same package is a full MCP client. Serve server.py over HTTP:

uv run mcp run server.py --transport streamable-http


then point a Client at it:

import asyncio

from mcp import Client


async def main() -> None:
    async with Client("http://localhost:8000/mcp") as client:
        result = await client.call_tool("add", {"a": 1, "b": 2})
        print(result.structured_content)  # {'result': 3}


asyncio.run(main())


A URL means Streamable HTTP, the transport you deploy. Client can also launch a local server as a stdio subprocess or take any custom transport; Clients has the rest.

Contributing

We are passionate about supporting contributors of all levels of experience and would love to see you get involved in the project. See the contributing guide to get started.

License

This project is licensed under the MIT License. See the LICENSE file for details.

프로젝트 링크
Data verified by PyPI on Oct 2, 2026 
Issues
Repository
Documentation
Homepage
Key dates
PyPI data
Released: 2026년 10월 3일
최신 릴리스
4 maintainers
PyPI data
dsp
jspahrsummers
Kludex
+1 more
Credits

작성자: Model Context Protocol a Series of LF Projects, LLC.

관리자: David Soria Parra

깃헙 통계
Data verified by PyPI on Oct 2, 2026
 저장소
 Stars: 24500
 Forks: 4016
 Open issues: 242
 Open PRs: 106
License

MIT License (MIT)

Requires

Python >=3.10

Provides Extra

cli rich

태그

automation llm mcp

분류
Development Status
5 - Production/Stable
Intended Audience
Developers
License
OSI Approved :: MIT License
Programming Language
Python :: 3
Python :: 3.10
Python :: 3.11
Python :: 3.12
Python :: 3.13
Python :: 3.14
프로젝트를 멀웨어로 신고
PyPI

Developed and maintained by the Python Software Foundation and Python community, for the Python community.

상태: All Systems Operational

오늘 바로 후원하기!
도움말
패키지 설치하기
패키지 업로드하기
사용자 가이드
프로젝트명 유지
자주 하는 질문
PyPI에 대하여
PyPI 블로그
인프라스트럭쳐 대시보드
통계
로고 & 트레이드마크
후원사
PyPI에 기여하기
버그와 피드백
GitHub에서 기여하기
PyPI 번역
PyPI 후원자
개발자 목록
PyPI 사용하기
이용 약관
보안 문제점 제보
행동 강령
개인정보 보호 공지
허용가능한 사용 정책
Anthropic, PBC
Visionary sponsor
Bloomberg
Visionary sponsor
Hudson River Trading
Visionary sponsor
Meta
Visionary sponsor
NVIDIA
Visionary sponsor
Microsoft
Sustainability sponsor
Depot
Continuous Integration
AWS
Cloud computing and Security Sponsor
Datadog
Monitoring
Fastly
CDN
Google
Download Analytics
Sentry
Error logging
StatusPage
Status page
"PyPI", "Python Package Index", 블록 로고는 pypi.org와 관련된 제품 및 서비스에 대한 Python Software Foundation의 등록 상표입니다. © 2026 Python Software Foundation 사이트맵 Deployed from b9dd37f
```

## q1-x1 · exit 127 · 0.3s

```
$ set -euo pipefail
export PATH="$HOME/.local/bin:$PATH"
E=/work/evidence2; mkdir -p "$E"; cd /work
echo "== /work 구조 (depth 3, .venv/node_modules/.git/__pycache__/evidence2 제외) =="
find /work -maxdepth 3 -name .venv -prune -o -name node_modules -prune -o -name .git -prune -o -name __pycache__ -prune -o -name evidence2 -prune -o -print | sort
EXC=(--exclude-dir=.venv --exclude-dir=node_modules --exclude-dir=site-packages --exclude-dir=evidence2 --exclude-dir=.git)
echo "== sales_summary 가 들어 있는 .py =="
grep -rln --include='*.py' "${EXC[@]}" sales_summary /work || echo "(없음)"
SERVER_PY=$( (grep -rln --include='*.py' "${EXC[@]}" sales_summary /work || true) | (xargs -r grep -lE 'FastMCP|MCPServer|from mcp|import mcp' || true) | head -n1 )
if [ -z "$SERVER_PY" ]; then echo "!! MCP 서버 파일을 찾지 못했습니다"; exit 1; fi
PROJ_DIR=""; d=$(dirname "$SERVER_PY")
while [ "$d" != "/" ]; do if [ -f "$d/pyproject.toml" ]; then PROJ_DIR=$d; break; fi; d=$(dirname "$d"); done
[ -n "$PROJ_DIR" ] || PROJ_DIR=$(dirname "$SERVER_PY")
if [ -x "$PROJ_DIR/.venv/bin/python" ]; then PY_CMD="$PROJ_DIR/.venv/bin/python"; else PY_CMD="uv run --project $PROJ_DIR python"; fi
echo "== .xlsx 파일 =="
mapfile -t XL < <(find /work -name .venv -prune -o -name node_modules -prune -o -name .git -prune -o -name evidence2 -prune -o -name '*.xlsx' -print | sort)
for f in "${XL[@]}"; do ls -l "$f"; done
XLSX=""
for f in "${XL[@]}"; do if grep -qF "$(basename "$f")" "$SERVER_PY"; then XLSX=$f; break; fi; done
[ -n "$XLSX" ] || XLSX=${XL[0]:-}
echo "SERVER_PY=$SERVER_PY"; echo "PROJ_DIR=$PROJ_DIR"; echo "PY_CMD=$PY_CMD"; echo "XLSX=$XLSX"
echo "== 서버 소스 =="; cat -n "$SERVER_PY"
echo "== pyproject.toml =="; cat "$PROJ_DIR/pyproject.toml" 2>/dev/null || echo "(없음)"
echo "== 버전 =="
$PY_CMD --version
$PY_CMD -c 'import importlib.metadata as m; [print(p, m.version(p)) for p in ("mcp", "openpyxl")]'
uv --version || true; node --version || true; claude --version || true
uname -a; nproc
echo "ANTHROPIC_API_KEY: $([ -n "${ANTHROPIC_API_KEY:-}" ] && echo set || echo unset)"
if [ -f "$HOME/.claude/.credentials.json" ]; then echo "~/.claude/.credentials.json 있음"; else echo "~/.claude/.credentials.json 없음"; fi
echo "== 등록된 MCP 서버 (~/.claude.json, .mcp.json) =="
rm -f "$E/mcp_reg.txt"
$PY_CMD - "$SERVER_PY" <<'PY'
import glob, json, os, sys
srv = sys.argv[1]; base = os.path.basename(srv); sdir = os.path.dirname(srv)
hits = []
def scan(servers, scope, cwd, src):
    for name, cfg in (servers or {}).items():
        print(f"[{scope}] src={src} cwd={cwd} name={name} type={cfg.get('type', 'stdio')} command={cfg.get('command')} args={cfg.get('args')} env_keys={sorted((cfg.get('env') or {}).keys())}")
        blob = json.dumps(cfg, ensure_ascii=False)
        if base in blob or sdir in blob:
            hits.append((scope, name, cwd))
p = os.path.expanduser("~/.claude.json")
try:
    d = json.load(open(p, encoding="utf-8"))
    scan(d.get("mcpServers"), "user", "/work", p)
    for proj, pc in (d.get("projects") or {}).items():
        scan(pc.get("mcpServers"), "local", proj, p)
except Exception as e:
    print(f"~/.claude.json 읽기 실패: {e!r}")
for f in glob.glob("/work/**/.mcp.json", recursive=True):
    if any(x in f for x in ("/.venv/", "/node_modules/", "/evidence2/")):
        continue
    try:
        scan(json.load(open(f, encoding="utf-8")).get("mcpServers"), "project", os.path.dirname(f), f)
    except Exception as e:
        print(f"{f} 읽기 실패: {e!r}")
hits.sort(key=lambda h: {"local": 0, "project": 1, "user": 2}[h[0]])
print("서버 파일과 매칭되는 등록:", hits)
with open("/work/evidence2/mcp_reg.txt", "w") as fh:
    if hits:
        fh.write(hits[0][1] + "|" + hits[0][2] + chr(10))
PY
MCP_NAME=""; MCP_CWD=/work
if [ -s "$E/mcp_reg.txt" ]; then IFS='|' read -r MCP_NAME MCP_CWD < "$E/mcp_reg.txt"; fi
[ -d "$MCP_CWD" ] || MCP_CWD=/work
{
  echo "SERVER_PY=$(printf %q "$SERVER_PY")"
  echo "PROJ_DIR=$(printf %q "$PROJ_DIR")"
  echo "PY_CMD=$(printf %q "$PY_CMD")"
  echo "XLSX=$(printf %q "$XLSX")"
  echo "MCP_NAME=$(printf %q "$MCP_NAME")"
  echo "MCP_CWD=$(printf %q "$MCP_CWD")"
} > "$E/env.sh"
echo "== env.sh =="; cat "$E/env.sh"
echo "== claude mcp list (cwd=$MCP_CWD) =="
(cd "$MCP_CWD" && timeout 120 claude mcp list) || echo "rc=$?"
== /work 구조 (depth 3, .venv/node_modules/.git/__pycache__/evidence2 제외) ==
/work
/work/claude_desktop_config.json
/work/inspector-lab.json
/work/logs
/work/logs/abs_uv.txt
/work/logs/bad_print_call.txt
/work/logs/bad_print_list.txt
/work/logs/bare_uv.txt
/work/logs/diff_print.txt
/work/logs/diff_relpath.txt
/work/logs/relpath_devdir.txt
/work/logs/relpath_tmp.txt
/work/logs/tools_list.json
/work/sales-mcp
/work/sales-mcp/.gitignore
/work/sales-mcp/.mcp.json
/work/sales-mcp/.python-version
/work/sales-mcp/README.md
/work/sales-mcp/bad_print.py
/work/sales-mcp/bad_relpath.py
/work/sales-mcp/pyproject.toml
/work/sales-mcp/sales.xlsx
/work/sales-mcp/server.py
/work/sales-mcp/src
/work/sales-mcp/src/sales_mcp
/work/sales-mcp/uv.lock
/work/scripts
/work/scripts/check_imports.py
/work/scripts/count_tokens.py
/work/scripts/make_configs.py
/work/scripts/make_data.py
/work/v1test
== sales_summary 가 들어 있는 .py ==
/work/sales-mcp/server.py
/work/sales-mcp/bad_relpath.py
/work/sales-mcp/bad_print.py
== .xlsx 파일 ==
-rw-r--r--. 1 root root 8422 Oct  7 07:03 /work/sales-mcp/sales.xlsx
SERVER_PY=/work/sales-mcp/server.py
PROJ_DIR=/work/sales-mcp
PY_CMD=uv run --project /work/sales-mcp python
XLSX=/work/sales-mcp/sales.xlsx
== 서버 소스 ==
     1	import sys
     2	from collections import defaultdict
     3	from pathlib import Path
     4	from openpyxl import load_workbook
     5	from mcp.server.mcpserver import MCPServer
     6	
     7	DATA = Path(__file__).resolve().parent / "sales.xlsx"
     8	mcp = MCPServer("sales")
     9	
    10	def load_rows(month: str) -> list[dict]:
    11	    ws = load_workbook(DATA, read_only=True).active
    12	    rows = ws.iter_rows(values_only=True)
    13	    header = next(rows)
    14	    return [dict(zip(header, r)) for r in rows if str(r[0]).startswith(month)]
    15	
    16	@mcp.tool()
    17	def sales_summary(month: str) -> dict:
    18	    """월별 매출 요약(총매출, 주문 수, 지역별 매출). month 는 'YYYY-MM' 형식 (예: 2026-09)."""
    19	    rows, by_region = load_rows(month), defaultdict(int)
    20	    for r in rows:
    21	        by_region[r["region"]] += r["amount"]
    22	    return {"month": month, "orders": len(rows), "total": sum(by_region.values()), "by_region": dict(by_region)}
    23	
    24	@mcp.tool()
    25	def top_products(month: str, n: int = 3) -> list[dict]:
    26	    """해당 월 매출 상위 n개 상품과 판매 수량, 매출."""
    27	    agg = defaultdict(lambda: {"qty": 0, "amount": 0})
    28	    for r in load_rows(month):
    29	        agg[r["product"]]["qty"] += r["qty"]
    30	        agg[r["product"]]["amount"] += r["amount"]
    31	    ranked = sorted(agg.items(), key=lambda kv: kv[1]["amount"], reverse=True)
    32	    return [{"product": p, **v} for p, v in ranked[:n]]
    33	
    34	if __name__ == "__main__":
    35	    print("sales MCP server starting", file=sys.stderr)
    36	    mcp.run()
== pyproject.toml ==
[project]
name = "sales-mcp"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "mcp>=2.3.0",
    "openpyxl>=3.1.5",
]

[project.scripts]
sales-mcp = "sales_mcp:main"

[build-system]
requires = ["uv_build>=0.12.23,<0.13.0"]
build-backend = "uv_build"
== 버전 ==
bash: line 25: uv: command not found

```

## q1-x2 · exit 1 · 0.2s

```
$ set -euo pipefail
export PATH="$HOME/.local/bin:$PATH"
source /work/evidence2/env.sh; E=/work/evidence2; cd /work
EXC=(--exclude-dir=.venv --exclude-dir=node_modules --exclude-dir=site-packages --exclude-dir=evidence2 --exclude-dir=.git)
echo "== seed/random 이 들어간 .py (데이터 생성 스크립트 후보) =="
for f in $( (grep -rlE --include='*.py' "${EXC[@]}" 'seed|random' /work || true) ); do echo "##### $f"; cat -n "$f"; done
echo "== 문서/로그에서 seed 언급 =="
grep -rnE "${EXC[@]}" --include='*.md' --include='*.txt' --include='*.sh' --include='*.log' --include='*.json' 'seed' /work || echo "(없음)"
echo "== 원본 xlsx =="; ls -l "$XLSX"; sha256sum "$XLSX"
echo "== '178000' / '178,000' 문자열 검색 (/work 텍스트 파일 전체) =="
grep -rnIE "${EXC[@]}" '178[,_]?000' /work || echo "(일치 없음)"
cat > "$E/inspect_xlsx.py" <<'PY'
import collections, datetime as dt, json, sys
from openpyxl import load_workbook
path, out = sys.argv[1], sys.argv[2]
wb = load_workbook(path)
print("시트:", wb.sheetnames, "| active:", wb.active.title)
for ws in wb.worksheets:
    print(f"--- 시트 '{ws.title}': dims={ws.dimensions} max_row={ws.max_row} max_col={ws.max_column}")
    for r in ws.iter_rows(min_row=1, max_row=4, values_only=True):
        print("   ", [f"{v!r}:{type(v).__name__}" for v in r])
ws = wb.active
rows = list(ws.iter_rows(values_only=True))
hdr = [str(h).strip() if h is not None else "" for h in rows[0]]
print("헤더:", hdr)
def find(keys, avoid=()):
    for i, h in enumerate(hdr):
        hl = h.lower()
        if any(k in hl for k in keys) and not any(x in hl for x in avoid):
            return i
    return None
di = find(["date", "날짜", "일자"])
ai = find(["amount", "금액", "매출", "sales", "revenue"], avoid=["unit"])
pi = find(["product", "상품", "품목", "제품", "item"])
qi = find(["qty", "quantity", "수량"])
ui = find(["price", "단가"])
print(f"열 추정: date={di} amount={ai} product={pi} qty={qi} unit_price={ui}")
if di is None or (ai is None and (qi is None or ui is None)):
    sys.exit("!! 날짜/금액 열을 특정하지 못함")
def to_date(v):
    if isinstance(v, dt.datetime): return v.date()
    if isinstance(v, dt.date): return v
    return dt.date.fromisoformat(str(v)[:10])
recs = []
for r in rows[1:]:
    if all(v is None for v in r):
        continue
    amt = r[ai] if ai is not None else r[qi] * r[ui]
    recs.append({"date": to_date(r[di]), "amount": amt, "product": r[pi] if pi is not None else None, "row": r})
print("데이터 행:", len(recs), "| 기간:", min(x["date"] for x in recs), "~", max(x["date"] for x in recs))
print("금액 타입:", dict(collections.Counter(type(x["amount"]).__name__ for x in recs)))
bym = collections.defaultdict(list)
for x in recs:
    bym[x["date"].strftime("%Y-%m")].append(x)
truth = {}
for m in sorted(bym):
    xs = bym[m]
    byp = collections.Counter(); cnt = collections.Counter()
    for x in xs:
        byp[x["product"]] += x["amount"]; cnt[x["product"]] += 1
    truth[m] = {"total": sum(x["amount"] for x in xs), "count": len(xs),
                "by_product": [[p, v, cnt[p]] for p, v in byp.most_common()],
                "max_row": max(x["amount"] for x in xs), "min_row": min(x["amount"] for x in xs)}
    print(f"  {m}: {len(xs)}건 합계 {truth[m]['total']:,}원")
print("== 2026-09 정답지 ==")
print(json.dumps(truth.get("2026-09"), ensure_ascii=False, indent=1, default=str))
target = 178000
print(f"== {target:,} 이 나올 수 있는 값/집계 탐색 ==")
found = []
for x in recs:
    for j, v in enumerate(x["row"]):
        if v == target:
            found.append(f"셀 값 (열 {hdr[j]}): {x['row']}")
for m, t in truth.items():
    for p, v, c in t["by_product"]:
        if v == target: found.append(f"{m} 상품 {p} 합계")
        if c and v / c == target: found.append(f"{m} 상품 {p} 평균")
    if t["total"] == target: found.append(f"{m} 월 합계")
    if t["count"] and round(t["total"] / t["count"], -3) == target: found.append(f"{m} 월 평균 {t['total'] / t['count']:,.1f} (천원 반올림 시 일치)")
    if t["max_row"] == target: found.append(f"{m} 최대 단일 거래")
    if t["min_row"] == target: found.append(f"{m} 최소 단일 거래")
allp = collections.Counter()
for x in recs: allp[x["product"]] += x["amount"]
for p, v in allp.items():
    if v == target: found.append(f"전체 기간 상품 {p} 합계")
if found:
    for f in found: print(f)
else:
    print("일치하는 값/집계 없음")
json.dump(truth, open(out, "w", encoding="utf-8"), ensure_ascii=False, indent=1, default=str)
print("정답지 저장:", out)
PY
$PY_CMD "$E/inspect_xlsx.py" "$XLSX" "$E/truth.json"
echo "== 생성 스크립트 재실행으로 재현성 확인 =="
GEN=$( (grep -rlE --include='*.py' "${EXC[@]}" 'seed' /work || true) | (xargs -r grep -lE 'xlsx|[.]save[(]' || true) | head -n1 )
if [ -z "$GEN" ]; then
  echo "생성 스크립트를 찾지 못함 → 독자 재현용 스크립트는 새로 써야 함"
else
  echo "GEN=$GEN"
  R="$E/regen"; rm -rf "$R"; mkdir -p "$R"
  cp -p "$XLSX" "$E/orig_backup.xlsx"
  before=$(sha256sum "$XLSX" | cut -d' ' -f1); mt_before=$(stat -c %Y "$XLSX")
  cp "$GEN" "$R/"
  (cd "$R" && $PY_CMD "$R/$(basename "$GEN")") || echo "재생성 rc=$?"
  after=$(sha256sum "$XLSX" | cut -d' ' -f1); mt_after=$(stat -c %Y "$XLSX")
  NEW=$(find "$R" -name '*.xlsx' | head -n1)
  if [ "$mt_before" != "$mt_after" ] || [ "$before" != "$after" ]; then
    echo "!! 스크립트가 원본 경로에 직접 저장함 (sha 동일 여부: $([ "$before" = "$after" ] && echo 동일 || echo 다름)) → 결과를 $R/overwritten.xlsx 로 옮기고 원본 복원"
    cp "$XLSX" "$R/overwritten.xlsx"; cp -p "$E/orig_backup.xlsx" "$XLSX"; NEW="$R/overwritten.xlsx"
  fi
  ls -la "$R"
  if [ -n "$NEW" ]; then
    sha256sum "$NEW"
    $PY_CMD - "$E/orig_backup.xlsx" "$NEW" <<'PY'
import sys
from openpyxl import load_workbook
a, b = (load_workbook(p) for p in sys.argv[1:3])
print("시트:", a.sheetnames, "vs", b.sheetnames)
same = a.sheetnames == b.sheetnames
for n in a.sheetnames:
    if n not in b.sheetnames:
        continue
    va = list(a[n].iter_rows(values_only=True)); vb = list(b[n].iter_rows(values_only=True))
    diff = sum(1 for x, y in zip(va, vb) if x != y) + abs(len(va) - len(vb))
    print(f"시트 {n}: 행 {len(va)} vs {len(vb)}, 다른 행 {diff}")
    same = same and diff == 0
print("결론:", "셀 값 완전 동일 → seed 고정으로 재현 가능" if same else "!! 값이 다름 → 재현 안 됨")
PY
  else
    echo "재생성된 xlsx 를 찾지 못함 (다른 경로에 저장했을 수 있음)"
  fi
fi
bash: line 3: /work/evidence2/env.sh: No such file or directory

```

## q1-x3 · exit 1 · 0.1s

```
$ set -euo pipefail
export PATH="$HOME/.local/bin:$PATH"
source /work/evidence2/env.sh; E=/work/evidence2
cat > "$E/rpc_client.py" <<'PY'
import argparse, json, os, selectors, statistics, subprocess, sys, tempfile, time

NL = bytes([10])
ap = argparse.ArgumentParser(description="MCP stdio 서버를 SDK 없이 JSON-RPC 로 직접 호출하고 시간을 잰다. 사용: rpc_client.py [옵션] -- 서버 실행 명령")
ap.add_argument("--cwd")
ap.add_argument("--repeat", type=int, default=1)
ap.add_argument("--calls", type=int, default=1)
ap.add_argument("--tool", default="sales_summary")
ap.add_argument("--args", help="툴 인자 JSON. 없으면 inputSchema 를 보고 2026-09 로 채움")
ap.add_argument("--xlsx", help="경로 인자가 있는 툴이면 이 값을 넣음")
ap.add_argument("--env", action="append", default=[])
ap.add_argument("--fd-check", help="호출 후 서버 프로세스가 이 파일 핸들을 몇 개 쥐고 있는지 셈")
ap.add_argument("--brief", action="store_true", help="initialize/tools/list 응답 출력 생략")
ap.add_argument("--protocol", default="2025-06-18")
argv = sys.argv[1:]
if "--" not in argv:
    sys.exit("사용법: rpc_client.py [옵션] -- 서버 실행 명령")
k = argv.index("--"); a = ap.parse_args(argv[:k]); cmd = argv[k + 1:]
print("[서버 명령]", " ".join(cmd), "| cwd:", a.cwd or os.getcwd())

def auto_args(schema):
    out = {}
    for key, s in (schema.get("properties") or {}).items():
        kl = key.lower(); t = s.get("type")
        if t is None and "anyOf" in s:
            t = next((x.get("type") for x in s["anyOf"] if x.get("type") != "null"), None)
        if any(w in kl for w in ("path", "file", "xlsx")):
            if a.xlsx: out[key] = a.xlsx
        elif "month" in kl or "period" in kl or kl in ("ym", "yyyymm"):
            out[key] = 9 if t == "integer" else "2026-09"
        elif "year" in kl:
            out[key] = 2026 if t == "integer" else "2026"
        elif key in (schema.get("required") or []):
            out[key] = 3 if t == "integer" else ""
    return out

def proc_tree(pid):
    pids = [pid]; i = 0
    while i < len(pids):
        p = pids[i]; i += 1
        try:
            for tid in os.listdir(f"/proc/{p}/task"):
                with open(f"/proc/{p}/task/{tid}/children") as fh:
                    pids += [int(x) for x in fh.read().split()]
        except OSError:
            pass
    return pids

def count_fds(pid, target):
    target = os.path.realpath(target); n = 0
    for p in proc_tree(pid):
        try:
            for fd in os.listdir(f"/proc/{p}/fd"):
                try:
                    if os.path.realpath(os.readlink(f"/proc/{p}/fd/{fd}")) == target: n += 1
                except OSError:
                    pass
        except OSError:
            pass
    return n

def run_once(verbose):
    env = os.environ.copy()
    for kv in a.env:
        kk, v = kv.split("=", 1); env[kk] = v
    errf = tempfile.TemporaryFile()
    t0 = time.perf_counter()
    p = subprocess.Popen(cmd, cwd=a.cwd, env=env, stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=errf)
    sel = selectors.DefaultSelector(); sel.register(p.stdout, selectors.EVENT_READ)
    buf = b""
    def send(o):
        p.stdin.write(json.dumps(o, ensure_ascii=False).encode() + NL); p.stdin.flush()
    def recv(mid, timeout=60):
        nonlocal buf
        end = time.time() + timeout
        while True:
            while NL in buf:
                line, buf = buf.split(NL, 1)
                if not line.strip():
                    continue
                try:
                    m = json.loads(line)
                except ValueError:
                    print("[stdout 비JSON]", line.decode(errors="replace")); continue
                if m.get("id") == mid:
                    return m
                if verbose:
                    print("[서버 메시지]", json.dumps(m, ensure_ascii=False))
            rem = end - time.time()
            if rem <= 0:
                raise TimeoutError(f"id={mid} 응답이 {timeout}s 동안 없음")
            if not sel.select(rem):
                continue
            chunk = os.read(p.stdout.fileno(), 65536)
            if not chunk:
                raise RuntimeError(f"서버 stdout 닫힘 (exit={p.wait(5)})")
            buf += chunk
    res = {"ok": False}
    try:
        send({"jsonrpc": "2.0", "id": 1, "method": "initialize", "params": {"protocolVersion": a.protocol, "capabilities": {}, "clientInfo": {"name": "rpc-probe", "version": "0"}}})
        m = recv(1); res["init_ms"] = (time.perf_counter() - t0) * 1000
        if verbose and not a.brief: print("[initialize 응답]", json.dumps(m, ensure_ascii=False))
        send({"jsonrpc": "2.0", "method": "notifications/initialized"})
        t = time.perf_counter(); send({"jsonrpc": "2.0", "id": 2, "method": "tools/list"}); m = recv(2)
        res["list_ms"] = (time.perf_counter() - t) * 1000
        tools = {x["name"]: x for x in m.get("result", {}).get("tools", [])}
        if verbose and not a.brief: print("[tools/list 응답]", json.dumps(m, ensure_ascii=False, indent=1))
        if a.tool not in tools:
            raise RuntimeError(f"툴 {a.tool} 없음: {list(tools)}")
        args = json.loads(a.args) if a.args else auto_args(tools[a.tool].get("inputSchema") or {})
        if verbose: print("[tools/call 인자]", json.dumps(args, ensure_ascii=False))
        res["call_ms"] = []
        for i in range(a.calls):
            t = time.perf_counter()
            send({"jsonrpc": "2.0", "id": 10 + i, "method": "tools/call", "params": {"name": a.tool, "arguments": args}})
            m = recv(10 + i); res["call_ms"].append((time.perf_counter() - t) * 1000)
            if i == 0:
                res["first_result"] = m
                if verbose: print("[tools/call 응답]", json.dumps(m, ensure_ascii=False, indent=1))
        if a.fd_check:
            res["fds"] = count_fds(p.pid, a.fd_check)
            print(f"[fd] tools/call {a.calls}회 후 서버 프로세스가 열고 있는 '{a.fd_check}' 핸들 수: {res['fds']}")
        res["ok"] = True
    except Exception as e:
        print(f"[실패] {type(e).__name__}: {e}")
    finally:
        try: p.stdin.close()
        except Exception: pass
        try: p.wait(5)
        except subprocess.TimeoutExpired:
            p.kill(); p.wait()
        errf.seek(0); err = errf.read().decode(errors="replace")
        if err.strip() and (verbose or not res["ok"]):
            print("[서버 stderr]"); print(err)
        print(f"[서버 종료코드] {p.returncode}")
    return res

results = [run_once(i == 0) for i in range(a.repeat)]
ok = [r for r in results if r["ok"]]
print("== 요약 ==")
for i, r in enumerate(results):
    if r["ok"]:
        c = r["call_ms"]
        warm = statistics.median(c[1:]) if len(c) > 1 else float("nan")
        print(f"#{i + 1} 기동~initialize {r['init_ms']:.0f}ms | tools/list {r['list_ms']:.1f}ms | 첫 tools/call {c[0]:.1f}ms | 이후 {len(c) - 1}회 중앙값 {warm:.1f}ms")
    else:
        print(f"#{i + 1} 실패")
if ok:
    print(f"기동~initialize 중앙값 {statistics.median(r['init_ms'] for r in ok):.0f}ms (n={len(ok)})")
    print(f"첫 tools/call 중앙값 {statistics.median(r['call_ms'][0] for r in ok):.1f}ms")
    warm = [x for r in ok for x in r["call_ms"][1:]]
    if warm: print(f"반복 tools/call 중앙값 {statistics.median(warm):.1f}ms (n={len(warm)})")
    fr = ok[0].get("first_result", {})
    print("첫 응답 isError:", fr.get("result", {}).get("isError"), "| JSON-RPC error:", fr.get("error"))
sys.exit(0 if ok and len(ok) == len(results) else 1)
PY
REL=$(realpath --relative-to="$PROJ_DIR" "$SERVER_PY")
echo "== 측정 환경 =="; uname -a; nproc; $PY_CMD --version
if [ -x "$PROJ_DIR/.venv/bin/python" ]; then
  echo "== A. 서버 단독: venv python 으로 직접 기동 (기동 5회 x 기동당 tools/call 10회, npx/Inspector 없음) =="
  $PY_CMD "$E/rpc_client.py" --cwd "$PROJ_DIR" --repeat 5 --calls 10 --fd-check "$XLSX" -- "$PROJ_DIR/.venv/bin/python" "$SERVER_PY" || echo "rc=$?"
else
  echo "== A. 건너뜀: $PROJ_DIR/.venv/bin/python 없음 =="
fi
echo "== B. uv run --directory 로 기동 (claude mcp add 에 쓰는 방식) =="
$PY_CMD "$E/rpc_client.py" --repeat 5 --calls 10 --fd-check "$XLSX" -- uv run --directory "$PROJ_DIR" python "$REL" || echo "rc=$?"
bash: line 3: /work/evidence2/env.sh: No such file or directory

```

## q1-x4 · exit 1 · 0.1s

```
$ set -euo pipefail
export PATH="$HOME/.local/bin:$PATH"
source /work/evidence2/env.sh; E=/work/evidence2
cat > "$E/make_messy.py" <<'PY'
import datetime as dt, os, shutil, sys
from openpyxl import load_workbook
from openpyxl.utils import get_column_letter

orig, outdir = sys.argv[1], sys.argv[2]
base = os.path.basename(orig)
KO = {"date": "날짜", "product": "상품명", "amount": "금액", "qty": "수량", "quantity": "수량", "region": "지역", "unit_price": "단가", "price": "단가", "category": "분류", "customer": "거래처", "channel": "채널", "id": "번호", "order_id": "주문번호", "manager": "담당자", "rep": "담당자"}

def load():
    wb = load_workbook(orig); ws = wb.active
    return wb, ws, [c.value for c in ws[1]]

def col(hdr, keys, avoid=()):
    for i, h in enumerate(hdr):
        hl = str(h).lower() if h is not None else ""
        if any(k in hl for k in keys) and not any(x in hl for x in avoid):
            return i + 1
    raise SystemExit(f"열을 찾지 못함: {keys} / 헤더 {hdr}")

def save(wb, name, desc):
    d = os.path.join(outdir, name); os.makedirs(d, exist_ok=True)
    wb.save(os.path.join(d, base)); print(f"[{name}] {desc}")

def as_date(v):
    if isinstance(v, dt.datetime): return v.date()
    if isinstance(v, dt.date): return v
    return dt.date.fromisoformat(str(v)[:10])

def dates_to_datetime(ws, D, n):
    for i in range(2, n + 1):
        v = ws.cell(i, D).value
        if isinstance(v, str):
            ws.cell(i, D).value = dt.datetime.fromisoformat(v[:10]); ws.cell(i, D).number_format = "yyyy-mm-dd"

def korean(ws, hdr):
    for j, h in enumerate(hdr, 1):
        ws.cell(1, j).value = KO.get(str(h).strip().lower(), h)

os.makedirs(os.path.join(outdir, "00_clean"), exist_ok=True)
shutil.copy(orig, os.path.join(outdir, "00_clean", base)); print("[00_clean] 원본 그대로 (대조군)")
wb, ws, hdr = load()
D = col(hdr, ["date", "날짜", "일자"]); A = col(hdr, ["amount", "금액", "매출"], ["unit"]); P = col(hdr, ["product", "상품", "품목"])
n = ws.max_row; L = get_column_letter(A)
print(f"원본: 시트 '{ws.title}', 헤더 {hdr}, 데이터 {n - 1}행, date열={D} amount열={A} product열={P}, 날짜 셀 타입={type(ws.cell(2, D).value).__name__}, 금액 셀 타입={type(ws.cell(2, A).value).__name__}")
sep_row = next((i for i in range(2, n + 1) if as_date(ws.cell(i, D).value).strftime("%Y-%m") == "2026-09"), 2)

wb, ws, hdr = load(); ws.insert_rows(20, amount=2)
save(wb, "01_blank_rows", "20행 위치에 완전히 빈 행 2개 삽입")

wb, ws, hdr = load(); old = ws.cell(sep_row, A).value; ws.cell(sep_row, A).value = None
save(wb, "02_amount_blank", f"{sep_row}행(2026-09 거래) 금액 셀만 비움, 원래 값 {old!r} → 정상이라면 9월 합계가 이만큼 줄어야 함")

wb, ws, hdr = load(); ws.cell(n + 1, D).value = "합계"; ws.cell(n + 1, A).value = f"=SUM({L}2:{L}{n})"
save(wb, "03_total_formula", f"맨 아래 합계 행: 날짜 칸 '합계', 금액 칸 수식 =SUM({L}2:{L}{n})")

wb, ws, hdr = load(); total = sum(ws.cell(i, A).value or 0 for i in range(2, n + 1)); ws.cell(n + 1, D).value = "합계"; ws.cell(n + 1, A).value = total
save(wb, "04_total_value", f"맨 아래 합계 행: 날짜 칸 '합계', 금액 칸 값 {total:,}")

wb, ws, hdr = load()
if isinstance(ws.cell(2, D).value, str):
    dates_to_datetime(ws, D, n); save(wb, "05_dates_datetime", "날짜 문자열 → 엑셀 날짜 셀(datetime, 표시형식 yyyy-mm-dd)")
else:
    for i in range(2, n + 1):
        ws.cell(i, D).value = as_date(ws.cell(i, D).value).strftime("%Y.%m.%d")
    save(wb, "05_dates_text_dots", "엑셀 날짜 셀 → '2026.09.03' 형태 문자열")

wb, ws, hdr = load(); korean(ws, hdr)
save(wb, "06_korean_headers", f"헤더 한글화: {[c.value for c in ws[1]]}")

wb, ws, hdr = load(); ws.insert_rows(1, amount=2); ws["A1"] = "2026년 매출 현황 (단위: 원)"
save(wb, "07_header_offset", "헤더 위에 제목 행 + 빈 행 (헤더가 3행)")

wb, ws, hdr = load(); c = wb.create_sheet("표지", 0); c["A1"] = "영업팀 월간 매출 보고"; c["A2"] = "작성: 영업지원팀"; wb.active = 0
save(wb, "08_cover_sheet_first", f"첫 시트(active)가 표지, 데이터는 두 번째 시트 '{ws.title}'")

wb, ws, hdr = load()
for i in range(2, n + 1):
    v = ws.cell(i, A).value
    ws.cell(i, A).value = f"{v:,}" if isinstance(v, (int, float)) else v
save(wb, "09_amount_text_comma", "금액이 '12,000' 같은 텍스트")

wb, ws, hdr = load()
dates_to_datetime(ws, D, n); korean(ws, hdr)
ws.insert_rows(40, amount=1)
ws.cell(ws.max_row + 1, D).value = "합계"
ws.insert_rows(1, amount=2); ws["A1"] = "2026년 매출 현황 (단위: 원)"
last = ws.max_row
ws.cell(last, A).value = f"=SUM({L}4:{L}{last - 1})"
c = wb.create_sheet("표지", 0); c["A1"] = "영업팀 월간 매출 보고"; wb.active = 0
save(wb, "10_realistic_combo", "표지 시트 + 제목 행 + 한글 헤더 + 날짜 셀 + 중간 빈 행 + 합계 수식 행")
PY
cat > "$E/robust_reader.py" <<'PY'
"""실무 엑셀 방어용 읽기 함수 (서버에 붙여 넣을 후보)."""
import datetime as dt, os, re, sys
from collections import Counter
from openpyxl import load_workbook

ALIASES = {
    "date": ("date", "날짜", "일자", "거래일", "판매일"),
    "product": ("product", "상품", "상품명", "품목", "제품"),
    "amount": ("amount", "금액", "매출", "매출액", "판매금액"),
}
SKIP_LABEL = re.compile("합계|총계|소계|total", re.I)

def _find_header(ws, scan_rows=10):
    for r, row in enumerate(ws.iter_rows(max_row=scan_rows, values_only=True), start=1):
        cells = [str(c).strip().lower() if c is not None else "" for c in row]
        idx = {}
        for key, names in ALIASES.items():
            for j, c in enumerate(cells):
                if c in names:
                    idx[key] = j; break
        if {"date", "amount"} <= idx.keys():
            return r, idx
    return None, None

def _to_date(v):
    if isinstance(v, dt.datetime): return v.date()
    if isinstance(v, dt.date): return v
    if isinstance(v, str):
        s = v.strip().replace(".", "-").replace("/", "-")
        try: return dt.date.fromisoformat(s[:10])
        except ValueError: return None
    return None

def _to_amount(v):
    if isinstance(v, bool): return None
    if isinstance(v, (int, float)): return v
    if isinstance(v, str):
        s = v.strip().replace(",", "").replace("원", "")
        if not s or s.startswith("="): return None
        try: return float(s) if "." in s else int(s)
        except ValueError: return None
    return None

def read_sales(path):
    wb = load_workbook(path, read_only=True, data_only=True)
    try:
        for ws in wb.worksheets:
            header_row, idx = _find_header(ws)
            if header_row:
                break
        else:
            raise ValueError("날짜/금액 헤더가 있는 시트를 찾지 못했습니다")
        rows, skipped = [], Counter()
        width = max(idx.values()) + 1
        for row in ws.iter_rows(min_row=header_row + 1, values_only=True):
            if not any(c not in (None, "") for c in row):
                skipped["빈 행"] += 1; continue
            if any(isinstance(c, str) and SKIP_LABEL.search(c) for c in row):
                skipped["합계 행"] += 1; continue
            row = tuple(row) + (None,) * (width - len(row))
            d, a = _to_date(row[idx["date"]]), _to_amount(row[idx["amount"]])
            if d is None or a is None:
                skipped["날짜·금액 불량"] += 1; continue
            rows.append({"date": d, "amount": a, "product": row[idx["product"]] if "product" in idx else None})
        return ws.title, header_row, rows, skipped
    finally:
        wb.close()  # read_only 모드는 파일 핸들을 계속 쥐고 있으므로 반드시 닫는다

if __name__ == "__main__":
    base = None
    for p in sys.argv[1:]:
        label = os.path.basename(os.path.dirname(p))
        try:
            sheet, hr, rows, skipped = read_sales(p)
        except Exception as e:
            print(f"{label:22} 실패: {type(e).__name__}: {e}"); continue
        sep = [r for r in rows if (r["date"].year, r["date"].month) == (2026, 9)]
        total = sum(r["amount"] for r in sep)
        byp = Counter()
        for r in sep: byp[r["product"]] += r["amount"]
        top3 = byp.most_common(3)
        if base is None: base = (total, top3)
        mark = "대조군" if label.startswith("00") else ("대조군과 일치" if (total, top3) == base else "대조군과 다름")
        print(f"{label:22} 시트={sheet} 헤더행={hr} 읽은행={len(rows)} 건너뜀={dict(skipped)} | 2026-09 합계={total:,} 건수={len(sep)} top3={top3} → {mark}")
PY
cat > "$E/fd_leak.py" <<'PY'
import gc, os, sys
from openpyxl import load_workbook
p = os.path.realpath(sys.argv[1])
def open_fds():
    n = 0
    for fd in os.listdir("/proc/self/fd"):
        try:
            if os.path.realpath(os.readlink(f"/proc/self/fd/{fd}")) == p: n += 1
        except OSError:
            pass
    return n
print("시작:", open_fds())
wb = load_workbook(p, read_only=True); cnt = sum(1 for _ in wb.active.iter_rows(values_only=True))
print(f"read_only 로드 + 전체 {cnt}행 순회 후 (close 전):", open_fds())
wb.close(); print("wb.close() 후:", open_fds())
def leaky():
    wb = load_workbook(p, read_only=True)
    return sum(1 for _ in wb.active.iter_rows(values_only=True))
for _ in range(5): leaky()
print("함수 안에서 열고 close 없이 5번 호출 후:", open_fds())
gc.collect(); print("gc.collect() 후:", open_fds())
wb2 = load_workbook(p); print("일반 모드(read_only=False) 로드 직후:", open_fds())
PY
XB=$(basename "$XLSX"); SB=$(basename "$SERVER_PY"); SDIR=$(dirname "$SERVER_PY")
RELX=$(realpath --relative-to="$SDIR" "$XLSX")
echo "== 현재 서버 코드에서 엑셀을 읽는 줄 =="
grep -nE 'load_workbook|active|close|iter_rows|worksheets|environ|getenv|xlsx|sheet' "$SERVER_PY" || echo "(없음)"
ENVVARS=$($PY_CMD - "$SERVER_PY" <<'PY'
import re, sys
s = open(sys.argv[1], encoding="utf-8").read()
q = "['" + chr(34) + "]"
names = set(re.findall("(?:getenv|environ[.]get|environ[[])[(]?[ ]*" + q + "([A-Za-z0-9_]+)" + q, s))
print(" ".join(sorted(n for n in names if re.search("XLSX|EXCEL|FILE|PATH|DATA|SALES", n, re.I))))
PY
)
echo "엑셀 경로 환경변수 후보: ${ENVVARS:-(없음)}"
echo "== 지저분한 엑셀 변형 만들기 =="
M="$E/messy"; rm -rf "$M" "$E/messy_run"
$PY_CMD "$E/make_messy.py" "$XLSX" "$M"
echo "== [1] 현재 서버 코드 그대로, 각 변형 파일로 sales_summary 호출 (00_clean 이 원본과 같은 답이면 격리 OK) =="
for V in $(ls "$M"); do
  echo; echo "################ $V ################"
  T="$E/messy_run/$V"; mkdir -p "$T"
  cp "$SDIR"/*.py "$T/"
  cp "$M/$V/$XB" "$T/$XB"
  case "$RELX" in ../*) ;; *) mkdir -p "$T/$(dirname "$RELX")"; cp "$M/$V/$XB" "$T/$RELX" ;; esac
  sed -i "s#$XLSX#$T/$XB#g" "$T/$SB"
  ENVARGS=(); for k in $ENVVARS; do ENVARGS+=(--env "$k=$T/$XB"); done
  if [ -x "$PROJ_DIR/.venv/bin/python" ]; then SRV=("$PROJ_DIR/.venv/bin/python" "$T/$SB"); else SRV=(uv run --project "$PROJ_DIR" python "$T/$SB"); fi
  $PY_CMD "$E/rpc_client.py" --brief --cwd "$T" --xlsx "$T/$XB" "${ENVARGS[@]}" -- "${SRV[@]}" || echo ">> rpc_client rc=$?"
done
echo; echo "== [2] 방어 코드 (robust_reader.py) 전문 =="; cat -n "$E/robust_reader.py"
echo "== [2] 방어 코드로 같은 변형 읽기 =="
$PY_CMD "$E/robust_reader.py" $(for V in $(ls "$M"); do echo "$M/$V/$XB"; done)
echo "== [3] read_only 워크북을 close 하지 않을 때 파일 핸들 =="
$PY_CMD "$E/fd_leak.py" "$XLSX"
bash: line 3: /work/evidence2/env.sh: No such file or directory

```

## q1-x5 · exit 1 · 0.2s

```
$ set -euo pipefail
export PATH="$HOME/.local/bin:$PATH"
source /work/evidence2/env.sh; E=/work/evidence2
cat > "$E/summarize_stream.py" <<'PY'
import json, re, sys
path, truth_path = sys.argv[1], sys.argv[2]
try:
    truth = json.load(open(truth_path, encoding="utf-8")).get("2026-09") or {}
except OSError:
    truth = {}
final = ""
for line in open(path, encoding="utf-8"):
    line = line.strip()
    if not line:
        continue
    try:
        m = json.loads(line)
    except ValueError:
        print("[비JSON]", line); continue
    t = m.get("type")
    if t == "system" and m.get("subtype") == "init":
        print("[init] model:", m.get("model"), "| permissionMode:", m.get("permissionMode"))
        print("[init] mcp_servers:", json.dumps(m.get("mcp_servers"), ensure_ascii=False))
        print("[init] MCP 툴:", [x for x in m.get("tools", []) if x.startswith("mcp__")])
    elif t == "assistant":
        for c in m.get("message", {}).get("content", []):
            if c.get("type") == "tool_use":
                print("[툴 호출]", c.get("name"), json.dumps(c.get("input"), ensure_ascii=False))
            elif c.get("type") == "text":
                print("[Claude 텍스트]", c.get("text"))
    elif t == "user":
        content = m.get("message", {}).get("content")
        if isinstance(content, list):
            for c in content:
                if isinstance(c, dict) and c.get("type") == "tool_result":
                    print("[툴 결과] is_error:", c.get("is_error"), json.dumps(c.get("content"), ensure_ascii=False))
    elif t == "result":
        final = m.get("result") or ""
        print("[결과] subtype:", m.get("subtype"), "| is_error:", m.get("is_error"), "| turns:", m.get("num_turns"), "| duration_ms:", m.get("duration_ms"), "| cost_usd:", m.get("total_cost_usd"))
        print("[결과] usage:", json.dumps(m.get("usage"), ensure_ascii=False))
        print("[결과] permission_denials:", json.dumps(m.get("permission_denials"), ensure_ascii=False))
        print("[최종 답변]"); print(final)
if truth and final:
    nums = {int(x.replace(",", "")) for x in re.findall("[0-9][0-9,]*[0-9]", final) if x.replace(",", "").isdigit()}
    print("== 정답지(2026-09) 대조 ==")
    print(f"월 합계 {truth['total']:,}:", "답변에 있음" if truth["total"] in nums else "답변에 없음")
    for p, v, c in truth["by_product"][:3]:
        print(f"상위 {p} {v:,} ({c}건):", "답변에 있음" if v in nums else "답변에 없음")
PY
cat > "$E/pty_claude.py" <<'PY'
import fcntl, os, pty, re, select, signal, struct, sys, termios, time
prompt, rawlog = sys.argv[1], sys.argv[2]
ROWS, COLS = 50, 140
ESC = chr(27); BEL = chr(7)
ANSI = re.compile(ESC + "[[][0-9;?]*[ -/]*[@-~]|" + ESC + "[]][^" + BEL + "]*" + BEL + "|" + ESC + "[()][0-9A-Za-z]|" + ESC + "[=>78]")
def strip(b):
    return ANSI.sub("", b.decode("utf-8", "replace")).replace(chr(13), chr(10))
try:
    import pyte
    screen = pyte.Screen(COLS, ROWS); stream = pyte.ByteStream(screen)
except ImportError:
    screen = None
pid, fd = pty.fork()
if pid == 0:
    os.environ.update(TERM="xterm-256color", COLUMNS=str(COLS), LINES=str(ROWS))
    os.execvp("claude", ["claude", prompt])
fcntl.ioctl(fd, termios.TIOCSWINSZ, struct.pack("HHHH", ROWS, COLS, 0, 0))
raw = bytearray(); recent = bytearray()
t0 = last_out = time.time(); last_action = 0.0
def snap(title):
    print(); print(f"===== 화면: {title} (t={time.time() - t0:.1f}s) =====")
    if screen:
        for l in screen.display:
            if l.strip(): print(l.rstrip())
    else:
        print(strip(bytes(recent)))
trusted = False; approvals = 0
while time.time() - t0 < 300:
    r, _, _ = select.select([fd], [], [], 0.5)
    if r:
        try:
            data = os.read(fd, 65536)
        except OSError:
            break
        if not data:
            break
        raw += data; recent += data; last_out = time.time()
        if screen: stream.feed(data)
        continue
    now = time.time()
    if now - last_out < 1.0 or now - last_action < 3:
        continue
    s = strip(bytes(recent))
    if not trusted and re.search("trust the files|Do you trust", s, re.I):
        snap("폴더 신뢰 확인 → Enter"); os.write(fd, chr(13).encode()); trusted = True; last_action = now; recent.clear(); continue
    if approvals < 3 and re.search("Do you want to proceed|Do you want to allow|Allow this", s):
        approvals += 1; snap(f"툴 호출 승인 프롬프트 #{approvals} → Enter (1. Yes)"); os.write(fd, chr(13).encode()); last_action = now; recent.clear(); continue
    if approvals and now - last_out > 25:
        snap("승인 후 응답 출력, 25초간 추가 출력 없음 → 종료"); break
    if not approvals and now - last_out > 90:
        snap("90초간 출력 없음, 승인 프롬프트 미검출 → 종료"); break
else:
    snap("300초 제한 도달")
snap("최종 화면")
try:
    os.kill(pid, signal.SIGTERM); time.sleep(1); os.kill(pid, signal.SIGKILL)
except ProcessLookupError:
    pass
try:
    os.waitpid(pid, 0)
except ChildProcessError:
    pass
open(rawlog, "wb").write(raw)
print(); print(f"===== 전체 출력 (ANSI 제거, 원본 {len(raw)} bytes 는 {rawlog}) =====")
print(strip(bytes(raw)))
PY
if [ -z "$MCP_NAME" ]; then echo "!! x1 에서 등록된 MCP 서버 이름을 못 찾음 → claude 세션 테스트 불가"; exit 1; fi
cd "$MCP_CWD"; echo "cwd=$(pwd) MCP_NAME=$MCP_NAME"
claude --version
timeout 120 claude mcp list || echo "rc=$?"
timeout 60 claude mcp get "$MCP_NAME" || echo "rc=$?"
run() {
  local label=$1 prompt=$2; shift 2
  echo; echo "======== [$label] claude -p '$prompt' $* ========"
  local rc=0
  timeout 300 claude -p "$prompt" --output-format stream-json --verbose "$@" > "$E/claude_$label.jsonl" 2> "$E/claude_$label.stderr" || rc=$?
  echo "exit=$rc"
  echo "--- stream-json 원본 ---"; cat "$E/claude_$label.jsonl"
  echo "--- stderr ---"; cat "$E/claude_$label.stderr"
  echo "--- 요약 ---"; $PY_CMD "$E/summarize_stream.py" "$E/claude_$label.jsonl" "$E/truth.json" || echo "요약 실패 rc=$?"
}
run p1_no_permission "2026년 9월 매출 요약해줘"
run p2_allow_summary "2026년 9월 매출 요약해줘" --allowedTools "mcp__${MCP_NAME}__sales_summary"
run p3_allow_top3 "2026년 9월 매출 상위 3개 알려줘" --allowedTools "mcp__${MCP_NAME}"
echo; echo "======== [interactive] 실제 TTY 세션: 승인 프롬프트 캡처 후 Enter(Yes) ========"
if uv run --no-project --with pyte python -c 'import pyte' >/dev/null 2>&1; then PTYPY="uv run --no-project --with pyte python"; else PTYPY="$PY_CMD"; echo "(pyte 사용 불가 → ANSI 제거 텍스트로 출력)"; fi
$PTYPY "$E/pty_claude.py" "2026년 9월 매출 요약해줘" "$E/claude_tty_raw.log" || echo "rc=$?"
bash: line 3: /work/evidence2/env.sh: No such file or directory

```

## q1-x6 · exit 1 · 0.2s

```
$ set -euo pipefail
export PATH="$HOME/.local/bin:$PATH"
source /work/evidence2/env.sh; E=/work/evidence2
REL=$(realpath --relative-to="$PROJ_DIR" "$SERVER_PY"); SB=$(basename "$SERVER_PY")
UVCMD=uv; if [ -x "$HOME/.local/bin/uv" ]; then UVCMD='${HOME}/.local/bin/uv'; fi
mk() { local dir=$1 json=$2; rm -rf "$dir"; mkdir -p "$dir/.claude"; echo "$json" > "$dir/.mcp.json"; echo '{"enableAllProjectMcpServers": true}' > "$dir/.claude/settings.local.json"; echo "--- $dir/.mcp.json"; cat "$dir/.mcp.json"; }
chk() { local dir=$1; shift; echo ">>> cd $dir; env $* claude mcp list"; (cd "$dir" && env "$@" timeout 120 claude mcp list) || echo "rc=$?"; echo ">>> claude mcp get sales-team"; (cd "$dir" && env "$@" timeout 60 claude mcp get sales-team) || echo "rc=$?"; }
D1="$E/scope_T1_envvar"
mk "$D1" "$(printf '{"mcpServers":{"sales-team":{"command":"%s","args":["run","--directory","${SALES_MCP_DIR}","python","%s"]}}}' "$UVCMD" "$REL")"
echo '## T1-a: ${SALES_MCP_DIR} (+ command 의 ${HOME}) 확장, 변수 설정함'; chk "$D1" SALES_MCP_DIR="$PROJ_DIR"
echo '## T1-b: 같은 .mcp.json, SALES_MCP_DIR 미설정'; chk "$D1" -u SALES_MCP_DIR
D2="$E/scope_T2_default"
mk "$D2" "$(printf '{"mcpServers":{"sales-team":{"command":"uv","args":["run","--directory","${SALES_MCP_DIR:-%s}","python","%s"]}}}' "$PROJ_DIR" "$REL")"
echo '## T2: ${VAR:-기본값} 문법, 변수 미설정'; chk "$D2" -u SALES_MCP_DIR
if [ -x "$PROJ_DIR/.venv/bin/python" ]; then
  D3="$E/scope_T3_relative"
  mk "$D3" "$(printf '{"mcpServers":{"sales-team":{"command":"%s","args":["./%s"]}}}' "$PROJ_DIR/.venv/bin/python" "$SB")"
  for f in "$(dirname "$SERVER_PY")"/*.py; do ln -sf "$f" "$D3/"; done
  ln -sf "$XLSX" "$D3/$(basename "$XLSX")"
  ls -la "$D3"
  echo '## T3-a: args 상대경로 ./server.py, .mcp.json 이 있는 프로젝트 루트에서 실행'; chk "$D3"
  mkdir -p "$D3/sub"
  echo '## T3-b: 같은 프로젝트의 하위 폴더에서 실행'; chk "$D3/sub"
else
  echo '## T3 건너뜀: venv python 없음'
fi
bash: line 3: /work/evidence2/env.sh: No such file or directory

```

## q1-x7 · exit 1 · 0.1s

```
$ set -euo pipefail
source /work/evidence2/env.sh; E=/work/evidence2; W="$E/win_json"; rm -rf "$W"; mkdir -p "$W"
cat > "$W/1_double_backslash.json" <<'EOF'
{
  "mcpServers": {
    "sales": {
      "command": "C:\\Users\\me\\.local\\bin\\uv.exe",
      "args": ["run", "--directory", "C:\\Users\\me\\sales-mcp", "python", "server.py"]
    }
  }
}
EOF
cat > "$W/2_single_backslash.json" <<'EOF'
{
  "mcpServers": {
    "sales": {
      "command": "C:\Users\me\.local\bin\uv.exe",
      "args": ["run", "--directory", "C:\Users\me\sales-mcp", "python", "server.py"]
    }
  }
}
EOF
cat > "$W/3_forward_slash.json" <<'EOF'
{
  "mcpServers": {
    "sales": {
      "command": "C:/Users/me/.local/bin/uv.exe",
      "args": ["run", "--directory", "C:/Users/me/sales-mcp", "python", "server.py"]
    }
  }
}
EOF
for f in "$W"/*.json; do echo "--- $f"; cat "$f"; done
$PY_CMD - "$W"/*.json <<'PY'
import json, sys
for p in sys.argv[1:]:
    name = p.rsplit("/", 1)[-1]
    try:
        cfg = json.load(open(p, encoding="utf-8"))["mcpServers"]["sales"]
        print(name, "→ 파싱 OK | command =", repr(cfg["command"]), "| args =", cfg["args"])
    except json.JSONDecodeError as e:
        print(name, "→ JSON 오류:", e)
PY
echo "(주의: Linux 의 JSON 파서로 문법만 확인한 것이고, 실제 Windows Claude Desktop 에서 실행한 결과는 아님)"
bash: line 2: /work/evidence2/env.sh: No such file or directory

```
