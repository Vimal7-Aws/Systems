MCP provides the standardized tool integration layer, 
    - Discovery
    - Schema conversion
    - Authentication
    - Transport

An MCP server exposes tools using the MCP protocol.
The MCP server exposes a tool definition containing:
- Tool name
- Description
- Input schema
- Tool execution capability

LangGraph provides the agent orchestration and execution layer.
    - Nodes
    - edges
    - router
    - memory
    - checkpointer
    - store
Parallel nodes
send
map-reduce
concurrency limits 
partial failure
result aggregation
cancellation
per tool timeout



The LangChain MCP adapter provides this integration.

3. MCP server discovery - Static versus dynamic discovery
4. The MCP protocol provides methods including:
- initialize
- tools/list
- tools/call
- resources/list
- resources/read
- prompts/list
- prompts/get
- Production recommendation: use a controlled MCP server registry rather than allowing an LLM to discover and connect to arbitrary MCP endpoints.

You need a tool governance strategy.
MCP Registry
|
+----------+----------+
|          |          |
v          v          v
Customer V1 Customer V2 Order V1
|          |          |
+----------+----------+
|
Tool Catalogue
|
v
LangGraph
Track:
- Server name.
- Server version.
- Tool name.
- Tool schema version.
- Tool description.
- Authentication requirements.
- Authorization policy.
- Compatibility status.
- Deprecation date.
- Owner.
- Health status.