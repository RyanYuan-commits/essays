```python
from mcp.server.fastmcp import FastMCP
from bilibili_api import search, sync
from typing import Any

mcp = FastMCP('BiliBili Mcp Server')

@mcp.tool()
def general_search(keyword : str) -> dict[Any, Any]:
    """
    Search BiliBili API with the given keyword.

    Args:
        keyword: The keyword to search for.

    Returns:
        dict: A dictionary containing the search results.
    """
    return sync(search.search(keyword))

if __name__ == '__main__':
    try:
        print('Mcp Server is running...')
        mcp.run(transport='stdio')
    except KeyboardInterrupt:
        print('Mcp Server stopped')
    except Exception as e:
        print(f'Error running Mcp Server: {e}')
```