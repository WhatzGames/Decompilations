# Decompilations
A collection of my decompilation projects

## Ghidra MCP Setup

This repository vendors both Ghidra and `ghidra-mcp` as submodules under
`tools/` so the MCP setup is reproducible and does not depend on the Ghidra
install under `~/bin/Ghidra`.

Pinned submodules:

- `tools/ghidra` -> `NationalSecurityAgency/ghidra` at `Ghidra_12.1.2_build`
- `tools/ghidra-mcp` -> `bethington/ghidra-mcp` at `v5.14.1`

After cloning this repo, initialize the submodules:

```bash
git submodule update --init --recursive
```

Build the submodule-provided Ghidra distribution:

```bash
cd tools/ghidra
GRADLE_USER_HOME="$PWD/.gradle-home" ./gradlew -I gradle/support/fetchDependencies.gradle
GRADLE_USER_HOME="$PWD/.gradle-home" ./gradlew buildGhidra
unzip -q -o build/dist/ghidra_12.1.2_DEV_*_linux_x86_64.zip -d build/dist/extracted
cd ../..
```

The extracted Ghidra root used by MCP is:

```text
tools/ghidra/build/dist/extracted/ghidra_12.1.2_DEV
```

Create a Python virtual environment for `ghidra-mcp` and install/deploy it
against the submodule-built Ghidra. Use `XDG_CONFIG_HOME` so deployment writes
to a repo-local Ghidra profile instead of `~/.config/ghidra`:

```bash
cd tools/ghidra-mcp
python3 -m venv .venv

GHIDRA_ROOT="../../tools/ghidra/build/dist/extracted/ghidra_12.1.2_DEV"
MCP_GHIDRA_CONFIG="../../tools/ghidra/build/mcp-user-config"

TOOLS_SETUP_BACKEND=gradle .venv/bin/python -m tools.setup preflight --ghidra-path "$GHIDRA_ROOT"
TOOLS_SETUP_BACKEND=gradle .venv/bin/python -m tools.setup ensure-prereqs --ghidra-path "$GHIDRA_ROOT"

XDG_CONFIG_HOME="$MCP_GHIDRA_CONFIG" \
TOOLS_SETUP_BACKEND=gradle \
.venv/bin/python -m tools.setup deploy --ghidra-path "$GHIDRA_ROOT"
cd ../..
```

If the MCP build cannot find a Java 21 toolchain, install or provide a JDK 21
and point Gradle at it:

```bash
JAVA_HOME=/path/to/jdk-21 \
PATH=/path/to/jdk-21/bin:$PATH \
GRADLE_OPTS=-Dorg.gradle.java.installations.paths=/path/to/jdk-21 \
XDG_CONFIG_HOME="$MCP_GHIDRA_CONFIG" \
TOOLS_SETUP_BACKEND=gradle \
.venv/bin/python -m tools.setup deploy --ghidra-path "$GHIDRA_ROOT"
```

Verify the headless server without opening the Ghidra GUI:

```bash
GHIDRA_ROOT="$PWD/tools/ghidra/build/dist/extracted/ghidra_12.1.2_DEV"
MCP_PROFILE="$PWD/tools/ghidra/build/mcp-user-config/ghidra/ghidra_12.1.2_PUBLIC"
MCP_JAR="$MCP_PROFILE/Extensions/GhidraMCP/lib/GhidraMCP-5.14.1.jar"
GHIDRA_CP="$MCP_JAR:$(find "$GHIDRA_ROOT/Ghidra" -name '*.jar' -print | paste -sd: -)"

XDG_CONFIG_HOME="$PWD/tools/ghidra/build/mcp-user-config" \
java -cp "$GHIDRA_CP" \
  -Dapplication.install.dir="$GHIDRA_ROOT" \
  com.xebyte.headless.GhidraMCPHeadlessServer \
  --bind 127.0.0.1 --port 18089
```

In another shell:

```bash
curl http://127.0.0.1:18089/check_connection
curl http://127.0.0.1:18089/health
curl http://127.0.0.1:18089/get_version
```

Expected responses include `GhidraMCP Headless Server v5.14.1-headless` and a
healthy JSON status. Stop the verification server when finished.
