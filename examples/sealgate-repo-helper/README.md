# Repo helper

Repo helper is a LangChain agent that answers questions about public GitHub repositories. It gets all its tools from DeepWiki (`https://mcp.deepwiki.com/mcp`) through a SealGate gateway. SealGate is a gateway that sits between the agent and its MCP servers. The admin of the SealGate org sees each tool call and controls the agent with policies.

This folder is a standalone example. It has its own virtual environment and does not use the dependencies of the template.

## Files

- `agent.py` is the agent. It loads `.env`, gets its tools from SealGate, and prints the answer.
- `sealgate-manifest.json` names the DeepWiki server and the access level of each tool.
- `list_tools.py` prints the tools of an MCP server: `python list_tools.py <server URL>`.
- `.env.example` shows the variables that `agent.py` and the `sealgate` command read.

## Set up

1. Make the virtual environment and install the packages:

   ```bash
   uv venv && . .venv/bin/activate
   uv pip install --no-deps --index-url https://test.pypi.org/simple/ "sealgate-test==0.2.3"
   uv pip install "httpx>=0.27,<1" "anyio>=4" "attrs>=23.2" "python-dateutil>=2.8" "mcp>=1.30,<2" "langchain>=1.0" "langchain-mcp-adapters>=0.3" "langchain-openai>=1.0"
   ```

   The SealGate SDK is a preview on TestPyPI. Install only that package from TestPyPI, with `--no-deps`, because anybody can upload a package there. Keep `mcp` below version 2, because `langchain-mcp-adapters` 0.3.1 does not work with `mcp` 2.x.

2. Copy `.env.example` to `.env`, fill in your SealGate token and your OpenRouter key, and run `chmod 600 .env`. The SealGate token comes from the agent builder of the SealGate developer portal. It expires after 24 hours.

3. Create the org and apply the manifest:

   ```bash
   set -a && . ./.env && set +a
   sealgate provision --org-name "<org name>" --manifest sealgate-manifest.json
   ```

   The command keeps the org in `.sealgate/org.json`.

4. Connect the agent to the org:

   ```bash
   sealgate connect --client-name "Repo helper" --no-browser
   ```

   The org admin opens the consent link, selects the org, and clicks Connect. If the browser cannot load the `http://127.0.0.1` page after that, copy the full address from the browser and run `sealgate connect --finish "<that address>"`.

5. Make sure that SealGate gives the agent the DeepWiki tools:

   ```bash
   sealgate tools
   ```

   The list must include `deepwiki_ask_wiki_question`, `deepwiki_read_wiki_contents`, and `deepwiki_read_wiki_structure`. If they are missing, run step 4 again and select the correct org.

`.env` and `.sealgate/` hold secrets. The `.gitignore` in this folder keeps them out of git.

## Run the agent

```bash
python agent.py "In langchain-ai/langgraph, what is a checkpointer?"
```

## Control the agent

The admin sees each tool call of the agent in the SealGate dashboard under Sessions. The admin adds policies under Advanced, then Access Control, then Policies. For example, this CEL expression with the action "block" limits DeepWiki to one repository:

```
resource.server == "deepwiki" && !(tool.args.repoName in ["langchain-ai/langgraph"])
```

In the manifest, `ask_wiki_question` has `write_operation: true`, because it sends free text to DeepWiki and free text can leak data. For the lethal-trifecta risk of a conversation, SealGate counts that tool as external communication.
