---
name: WxO Architecture Decision Report
about: Enable Dynamic Tool Fetching for Remote MCP Servers
title: 'wxO ADR: Enable Dynamic Tool Fetching to Support User-Specific Access Control for Remote MCP Servers'
labels: 'wo-adr'
assignees: ''

---
---
##### status: proposed
##### date: 2026-03-12
##### deciders: [To be updated]
---

# wxO ADR: Enable Dynamic Tool Fetching to Support User-Specific Access Control for Remote MCP Servers

## Context and Problem Statement

watsonX Orchestrate currently supports integration with remote MCP (Model Context Protocol) servers to extend agent capabilities with external tools. The current implementation follows a static tool registration approach where tools are fetched and registered during the MCP server registration phase.

### Current MCP Workflow

When a user registers a remote MCP server:

1. The remote MCP server is registered on the Context Forge gateway
2. MCP initialization occurs and `list_tools` is executed immediately
3. The tool list and schema are saved in Context Forge and the tools table
4. Tools are bound to the agent for subsequent invocations
5. Users can invoke these pre-registered tools through the agent

### The Problem

The static tool registration approach creates a **tool lock-in problem** based on the first user's access level:

- **User Access Variability**: MCP servers often implement user-specific access controls, where different users have access to different sets of tools based on their permissions, roles, or organizational policies
- **First User Lock-in**: The tool list is determined by the first user who registers the MCP server, effectively locking all subsequent users to that user's access level
- **Permission Mismatch**: Users with higher privileges cannot access tools they should have access to, while users with lower privileges may see tools they cannot actually use
- **Stale Tool Lists**: Tool availability changes on the MCP server are not reflected until manual refresh or re-registration
- **Multi-tenancy Issues**: In multi-tenant environments, this creates security and usability concerns as tool visibility is not properly isolated per user

**Example Scenario**: If User A (with basic access) registers an MCP server that provides 5 tools, and later User B (with admin access) who should have access to 10 tools tries to use the same MCP server, User B will only see the 5 tools that were available to User A.

## Decision Drivers

- **User-Specific Access Control**: Support dynamic tool availability based on individual user permissions and access levels
- **Eliminate Tool Lock-in**: Remove dependency on the first user's access level for tool registration
- **Real-time Tool Availability**: Ensure agents always have access to the most current tool list from MCP servers
- **Multi-tenancy Support**: Properly isolate tool visibility and access across different users and tenants
- **Security Compliance**: Enforce proper authorization boundaries for tool access
- **Operational Simplicity**: Reduce manual intervention for tool list updates and refreshes
- **Scalability**: Support growing number of users with varying permission levels

## Considered Options

| Options | Pros | Cons |
|---------|------|------|
| **Option 1: Static Tool Registration (Current)** | <p>• Simple implementation<p>• Lower runtime overhead<p>• Tools cached and immediately available<p>• Predictable performance | <p>• Tool lock-in based on first user<p>• No support for user-specific access<p>• Requires manual refresh for tool updates<p>• Security concerns in multi-tenant scenarios<p>• Stale tool information |
| **Option 2: Dynamic Tool Fetching at Runtime (Proposed)** | <p>• User-specific tool access<p>• No tool lock-in<p>• Always up-to-date tool list<p>• Better security and isolation<p>• No manual refresh needed<p>• Supports dynamic MCP server changes | <p>• Additional `list_tools` call at runtime<p>• Slightly higher latency per invocation<p>• Requires caching strategy for optimization<p>• More complex implementation |
| **Option 3: Hybrid with Periodic Refresh** | <p>• Balanced approach<p>• Reduced runtime calls<p>• Some level of freshness | <p>• Still has tool lock-in issue<p>• Refresh interval creates staleness window<p>• Doesn't solve user-specific access<p>• Complex refresh logic needed<p>• Partial solution only |

## Decision Outcome

We have decided to **implement dynamic tool fetching at runtime (Option 2)** for remote MCP servers in watsonX Orchestrate.

### Implementation Approach

#### 1. CLI Command Enhancement

Users will register MCP servers with a new `--dynamicToolFetching` flag:

```bash
orchestrate toolkits import \
  --kind mcp \
  --url "https://remote-mcp-server.example.com" \
  --transport "http_streamable" \
  --dynamicToolFetching --app-id “for connection verification only”
```

**Supported Transport Types**:
- `http_streamable`: HTTP-based streaming transport
- `sse`: Server-Sent Events transport

#### 2. Registration Behavior

When `--dynamicToolFetching` is enabled:

- **Connection Validation**: During registration, a lightweight connectivity check is performed:
  - Establish connection to the MCP server using provided URL and transport
  - Execute MCP initialization handshake to verify server is reachable and compatible
  - Optionally perform a test `list_tools` call to validate the server responds correctly (without storing results)
  - If validation fails, registration is rejected with appropriate error message
- The MCP server gateway is registered in the Toolkit API and Context Forge upon successful validation
- The toolkit metadata (URL, transport type, configuration) is stored
- **Tools are NOT listed or stored during import** - only the gateway registration occurs
- The agent is bound to the toolkit (MCP server gateway) with the `dynamicToolFetching` flag
- Individual tools remain unregistered until runtime discovery
- Server metadata and connection details are stored for runtime use

#### 3. Runtime Tool Discovery and Execution

When an agent needs to execute a task:

1. **Tool Discovery Phase**:
   - Agent invokes `list_tools` on the MCP server with the current user's context
   - User authentication credentials are passed to the MCP server
   - MCP server returns tools available for that specific user
   - Tool list is cached temporarily for the user session

2. **Tool Execution Phase**:
   - Agent selects appropriate tool(s) from the user-specific list
   - Tool invocation proceeds with user context
   - Results are returned to the agent

#### 4. Performance Optimization: Async Prefetching

To mitigate the latency impact of runtime tool listing:

- **Prefetch on Agent Initialization**: When a user initiates an agent session, trigger an asynchronous `list_tools` call in the background
- **Session-based Caching**: Cache the tool list for the duration of the user session
- **Cache Invalidation**: Implement TTL-based cache expiration (configurable, default: 5 minutes)
- **Lazy Loading**: If cache is stale or unavailable, fetch tools synchronously when needed

### Architecture Flow

```
User Request → Agent Runtime
                    ↓
            [Check Tool Cache]
                    ↓
         Cache Miss or Expired?
                    ↓
    [Invoke list_tools with User Context]
                    ↓
         MCP Server (User-Specific Tools)
                    ↓
         [Cache Tools for Session]
                    ↓
         [Select & Execute Tool]
                    ↓
              Return Result
```

### Consequences

#### Functional Consequences

**Advantages**:

1. **User-Specific Tool Access**: Each user sees and can access only the tools they are authorized to use based on their permissions
2. **No Tool Lock-in**: Tool availability is determined at runtime for each user, eliminating first-user dependency
3. **Always Current**: Tool lists are always up-to-date, reflecting real-time changes on the MCP server
4. **Better Multi-tenancy**: Proper isolation of tool visibility across users and tenants
5. **Reduced Operational Overhead**: No need for manual tool list refresh or re-registration
6. **Dynamic MCP Evolution**: MCP servers can add, remove, or modify tools without requiring re-registration

**Drawbacks**:

1. **Initial Latency**: First tool invocation in a session may have slightly higher latency due to tool listing
2. **Increased MCP Server Load**: More frequent `list_tools` calls to MCP servers (mitigated by caching)
3. **Cache Management Complexity**: Need to implement and maintain session-based caching logic

#### Non-functional Consequences

**Security**:
- ✅ **Improved**: Proper enforcement of user-level access controls
- ✅ **Improved**: Better tenant isolation in multi-tenant deployments
- ✅ **Improved**: Reduced risk of unauthorized tool access
- ⚠️ **Consideration**: User credentials must be securely passed to MCP servers

**Performance**:
- ⚠️ **Impact**: Additional network call for `list_tools` (typically 50-200ms)
- ✅ **Mitigation**: Async prefetching reduces perceived latency to near-zero
- ✅ **Mitigation**: Session-based caching eliminates repeated calls
- 📊 **Monitoring**: Track `list_tools` call latency and cache hit rates

**Scalability**:
- ✅ **Improved**: Better support for large user bases with varying permissions
- ⚠️ **Consideration**: MCP servers must handle increased `list_tools` traffic
- ✅ **Mitigation**: Caching reduces load on MCP servers

**Reliability**:
- ⚠️ **Dependency**: Agent execution now depends on MCP server availability for tool listing
- ✅ **Mitigation**: Implement fallback mechanisms and retry logic
- ✅ **Mitigation**: Cache provides resilience during temporary MCP server unavailability

**Monitoring and Observability**:
- 📊 **New Metrics Required**:
  - `list_tools` call frequency and latency
  - Cache hit/miss rates per user session
  - Tool availability per user
  - MCP server response times
- 📊 **Alerting**: Monitor for MCP server failures affecting tool discovery

**Operational**:
- ✅ **Simplified**: No manual tool refresh operations needed
- ✅ **Improved**: Better debugging with user-specific tool visibility
- ⚠️ **New Requirement**: Cache management and monitoring

### Migration Path

For existing MCP server registrations:

1. **Backward Compatibility**: Existing static registrations continue to work without changes
2. **Opt-in Migration**: Users can re-register with `--dynamicToolFetching` flag when ready
3. **Gradual Rollout**: Support both modes during transition period
4. **Documentation**: Provide migration guide for users to update their registrations

### Configuration Parameters

```yaml
mcp:
  dynamicToolFetching:
    enabled: true
    cache:
      ttl: 300  # seconds (5 minutes)
      maxSize: 1000  # maximum cached tool lists
    prefetch:
      enabled: true
      timeout: 5000  # milliseconds
    retry:
      maxAttempts: 3
      backoff: exponential
```

## More Information

### Related Decisions

- **Precursor**: [0501-remote-mcp-integration.md](https://github.ibm.com/WatsonOrchestrate/architecture/blob/mcp-gateway-integration-phase1/decisions/0501-remote-mcp-integration.md) - Initial remote MCP integration architecture
- **Related**: Multi-tenancy and user isolation requirements for watsonX Orchestrate

### Reference Links

- **Code Repository**: [wxo-server](https://github.ibm.com/WatsonOrchestrate/wxo-server)
- **MCP Specification**: Model Context Protocol documentation
- **Context Forge Gateway**: Integration documentation

### Future Considerations

1. **Smart Caching**: Implement predictive caching based on user behavior patterns
2. **Tool Recommendation**: Use historical data to pre-fetch likely-needed tools
3. **Batch Operations**: Support batch tool listing for multiple MCP servers
4. **Metrics Dashboard**: Build observability dashboard for dynamic tool fetching performance
5. **A/B Testing**: Compare performance metrics between static and dynamic approaches

### Open Questions

- What is the acceptable latency threshold for tool listing calls?
- Should cache TTL be configurable per MCP server or globally?
- How should we handle MCP servers that don't support user-specific tool listing?
- What fallback behavior should occur if `list_tools` fails during runtime?
- **Connection Validation Strategy**: Should the registration validation perform a full `list_tools` call or just a basic connectivity check? Trade-offs:
  - Full `list_tools`: Validates complete functionality but may expose first-user access level issue
  - Basic connectivity: Faster validation but doesn't guarantee tool listing will work at runtime
  - Recommended: Use basic MCP initialization handshake for validation, defer tool listing to runtime

---

**Document Version**: 1.0  
**Last Updated**: 2026-03-12  
**Status**: Proposed - Awaiting review and approval