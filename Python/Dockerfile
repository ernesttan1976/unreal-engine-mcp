FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app/Python

# Install dependencies from pyproject.toml
COPY pyproject.toml ./
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir .

# Copy source
COPY . ./

# Default to streamable HTTP for Docker. The endpoint will be:
#   http://localhost:8000/mcp
ENV UNREAL_MCP_TRANSPORT=streamable-http \
    FASTMCP_HOST=0.0.0.0 \
    FASTMCP_PORT=8000

EXPOSE 8000

CMD ["python", "unreal_mcp_server_advanced.py"]
