---
title: "Building an MCP Server"
date: 2026-05-04T08:21:19-05:00
description: "Learn to let agents loose on your resources"
---

# Introduction

Model Context Protocol is a standard method for connecting GenAI to external systems. It's widely adopted and tools like Claude and Gemini have comprehensive support for MCP. Building a MCP Server to handle requests from LLMs is a great first step when looking to integrate existing applications with GenAI or when building agents. I'll cover how to use [FastMCP](https://gofastmcp.com/getting-started/welcome) in this blog post, and some of the high level concepts involved. I'll touch on SAS Viya and our case management platform Visual Investigator, the lessons here would be generally applicable to any cloud based platform and CRUD app.

# FastMCP

This library is great for building an MCP server. It handles a lot of the plumbing involved when writing an MCP server, often handling any boilerplate entirely. To install use your package manager to install fastmcp: `uv add fastmcp`. To get started, here's what a simple MCP server looks like:
```python
from fastmcp import FastMCP

mcp = FastMCP("MyServer")

@mcp.tool
def greet(name: str) -> str:
    """Greet a user by name."""
    return f"Hello, {name}!"

if __name__ == "__main__":
    mcp.run()
```
and to run the server:
```bash
fastmcp run my_server.py:mcp --transport http --port 8000
```
Note: FastMCP doesn't use HTTP access by default so the --transport and --port options are required. 

The FastMCP object is really important, this is the main way to configure and define your MCP server. For instance, you can add an instructions string which will be provided to GenAI clients to help them understand what your server does and what it provides. The second thing to look at is `@mcp.tool`, this decorator makes the greet function externally available. Tools are functions provided by an MCP server.

When writing an MCP server think of tools as your interface, only register the functions which make sense for outside clients. Aim for tools that represent complete, meaningful operations like `edit_record` or `delete_record` rather than low-level primitives like `lock_record` or `get_field_value`. Exposing too many granular operations forces clients to orchestrate complex multi-step plans that are brittle and error-prone. Instead, encapsulate the necessary steps (like acquiring locks or retrieving specific fields) within higher-level tool implementations. This keeps the agent's reasoning simple and focused on business logic rather than implementation details. This is where we leave the science and get into the art of creating agents and GenAI implementations.

The code for the basic server can be found [here on GitHub](https://gist.github.com/mackles/a4dda53c2dc23b8cf533b60e9c160a06). 

# Authentication

You may be tempted to turn authentication off, because let's be honest it can be a little fiddly at times to get working. However, it's really important to make sure you're using authentication and authorization for your agents, not just from a security perspective but also to ensure that the clients using your MCP server can only make changes you've approved. Avoid giving broad access where possible.

The FastMCP documentation covers a lot of the authentication methods, including Dynamic Client Registration (the new and hip way to register MCP servers with applications). For the Viya MCP server I built, I used the [OAuthProxy](https://gofastmcp.com/servers/auth/oauth-proxy) methodology as this supports applications that don't support DCR, which is most enterprise software. 

First you have to register your MCP server as a client, which allows the MCP server to access your application on behalf of the user. [This post](https://developer.sas.com/rest-apis/SASLogon/docs/getting-started/authentication?cadence=Viya_35) documents the steps required for Viya. Most of the parameters you register the client with will be provided to the MCP server, so keep track of them.

A simple configuration for the proxy is:
```python
JWKS_URI = f"{endpoint}/SASLogon/token_keys"

token_verifier = JWTVerifier(jwks_uri=JWKS_URI, audience=[])

viya_auth = OAuthProxy(
    upstream_authorization_endpoint=f"{endpoint}/SASLogon/oauth/authorize",
    upstream_token_endpoint=f"{endpoint}/SASLogon/oauth/token",
    upstream_client_id="vi-mcp",
    upstream_client_secret="supersecret",
    jwt_signing_key="default",
    base_url="http://localhost:8000",
    forward_pkce=True,
    token_verifier=token_verifier,
    valid_scopes=["openid", "uaa.user"],
)
```

A lot of these parameters are standard and can be found in the documentation for the application you're integrating with. You'll define scopes which can subset the access the MCP server has to a users account. For instance, you can omit the admin scope to ensure that administrators using your MCP server don't inadvertently give GenAI clients enhanced permissions. The `jwt_signing_key` variable is an important one to read more about. The MCP server issues its own JSON Web Tokens for authentication and doesn't give the client (in my case Claude Code) the upstream token (the one Viya provides).

I registered the client with localhost as I'm running the MCP server on my laptop, others wanting to use your MCP server will have to run it locally too. This can be updated to the hostname of a virtual machine where you deploy the server in production. Also, in production you will not want to have the secrets hard coded in Python, often you'll want to store these in a secure key/value store like your clouds secrets manager. 

I recommend reading the [OAuthProxy](https://gofastmcp.com/servers/auth/oauth-proxy) section of the FastMCP documentation as this goes into more detail in how the authentication flow works. The main point is to be able to understand the client flow when you implement your own agent (Claude Code has already implemented this for instance) and the JWT boundaries to understand when you should use the upstream (Viya/application) or downstream(Claude/Agent) token. The concept of upstream/downstream tokens is to prevent clients who authenticate to the MCP server being able to call the upstream application directly. When we give our client access to the MCP server we don't want to give the client access to a token which gives a greater set of actions through the upstream service (e.g. all the Viya APIs).

```goat
    Claude / Agent              MCP Server               Viya / App
         |                          |                         |
         |   1. authenticate        |                         |
         |------------------------->|                         |
         |                          |                         |
         |   2. downstream JWT      |                         |
         |<-------------------------|                         |
         |                          |                         |
         |   3. tool call + JWT     |                         |
         |------------------------->|                         |
         |                     (exchanges                     |
         |                      for upstream   4. API call    |
         |                         token)-------------------->|
         |                                                    |
         |                          |                         |
         |                          |   5. response           |
         |                          |<------------------------|
         |   6. result only         |                         |
         |<-------------------------|                         |
         |                          |                         |
```

# Connecting Claude Code

I used Claude Code as my agent for this project. To register an MCP server with Claude, run:
```bash
claude mcp add --transport http vi http://localhost:8000/mcp
```

This will register the MCP server you're running locally as `vi` with Claude. Running Claude Code and entering the `/mcp` command will verify the MCP server is available. Claude will prompt you to follow the OAuth authentication flow and open a browser and redirect you back to your MCP server. 

# Writing tools

Now, to the fun bit. Let's write a tool for retrieving a record from Visual Investigator:

```python
@asynccontextmanager
async def authenticated_client():
    token = get_access_token().token
    async with httpx.AsyncClient(headers={"Authorization": f"Bearer {token}"}) as client:
        yield client

@mcp.tool
async def get_vi_page(page_type: str, entity_id: str) -> dict:
    """Fetch a SAS Visual Investigator page by page type and entity ID."""
    async with authenticated_client() as client:
        response = await client.get(f"{VI_BASE_URL}/pages/{page_type}/{entity_id}")
        response.raise_for_status()
        return response.json()
```

The `authenticated_client` function handles getting the upstream token (OAuth token) to access our application (Viya) using the `get_access_token()` function from FastMCP. If we do not do this the MCP server will try to pass along the JWT that the client is using to authenticate to the MCP server as discussed earlier. Remember, this step runs on the MCP server so that upstream OAuth token is not given to the client, maintaining proper separation.

The `get_vi_page` function is doing a pretty simple REST request and returning the response straight through to the client, there's two interesting things to note here. First, we can provide parameters as part of tools (in this case record type and record id) which gives clients parameters when using tools - cool! Secondly, we should really consider not returning the full JSON as this can cause confusion (e.g. for this we probably only want the records data to be returned and not the metadata) and a large JSON response can burn through our token budgets quickly. 


# Testing it out

Once you've written a couple of basic tools and confirmed the authentication works by registering your MCP server with Claude. The next step is to plan some things out, in my simple example I wrote a couple of tools to fetch metadata about the records in Visual Investigator. Let's try and populate a lab test record for a foodborne illness: 
```
❯ Hey Claude, give me a lab test for a foodborne illness using the vi tools                                                                                                                                         

⏺ Skill(/create-record)
  ⎿  Successfully loaded skill

⏺ I'll help you create a lab test record for a foodborne illness in SAS Visual Investigator. Let me start by exploring the available entity types.

  Called vi (ctrl+o to expand)

⏺ Perfect! I can see lab_test is available. Let me retrieve its metadata to understand the structure.

  Called vi (ctrl+o to expand)

⏺ Lab Test is the relevant entity type. This is designed to record diagnostic test results including disease type, test outcome, and patient information. For a foodborne illness test, I need to retrieve the
  reference data for disease type, test results, and other key fields.

  Let me get the reference lists for the important fields:

  Called vi 3 times (ctrl+o to expand)

⏺ Great! I found Salmonellosis in the disease_event list, which is a common foodborne illness. I'll create a lab test record for a positive Salmonella result with a stool specimen:

  Called vi (ctrl+o to expand)

⏺ Perfect! I've created a lab test record for a foodborne illness. Here are the details:

  Lab Test Created: LBT-2026-40

  - Disease: Salmonellosis (foodborne illness)
  - Patient: John Doe, age 42
  - Test Name: Stool Culture for Salmonella
  - Specimen Type: Stool
  - Result: Positive
  - Test Date: 2026-05-01
  - Lab: County Health Laboratory
  - Status: Awaiting Review
  - Created: 2026-05-01 at 15:50:47 UTC

  The record has been successfully stored in SAS Visual Investigator and can be linked to disease events, patients, or other related records as needed.

※ recap: We created a lab test record for Salmonellosis (LBT-2026-40) with a positive stool culture result for patient John Doe. The record is now stored in SAS Visual Investigator and ready for use. (disable
  recaps in /config)

```

It works! Some of the magic here is Claude Code being able to plan and execute effectively, but we have given it the ability to interact with a new system it couldn't have before. Claude has been able to progress through logical steps too, getting metadata on the record type to understand what fields are there and reference data values to ensure data entry is correct.

# Conclusion

This post has covered the basics of writing an MCP server, but there's some key areas we'd want to look at to productionize and improve our server. First, moving away from a dev deployment utilizing localhost and towards an MCP server co-located with the application. Secondly, we'd want to improve our tools to return data in a more condensed and clear format to avoid confusing agents and using excessive tokens. Lastly, we probably want to look into [skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) and other features to ensure our agents plan correctly, without these our agents can sometimes make unnecessary calls to tools. 

All in all building this didn't take more than an hour or two, FastMCP really does make it easy.

