# Agent Guidelines for Jarvis Monitor

## Build/Run Commands
- **Build image**: `docker-compose build` or `docker build -t jarvis-monitor .`
- **Run container**: `docker-compose up -d` (detached) or `docker-compose up` (foreground)
- **View logs**: `docker-compose logs -f` (follow mode)
- **Stop**: `docker-compose down`
- **Restart**: `docker-compose restart`
- **Test locally**: `python3 monitor.py` (requires Docker socket access and env vars)

## Code Style
- **Language**: Python 3.13
- **Imports**: Standard library first, then third-party (requests, docker). Use absolute imports.
- **Formatting**: 4 spaces indent. Max line length ~100 chars (not strict).
- **Docstrings**: Brief function docstrings in triple-quotes for main functions.
- **Naming**: `snake_case` for functions/variables, `UPPER_CASE` for constants.
- **Error Handling**: Use try-except blocks; print errors to `sys.stderr` with emoji prefixes (❌, ⚠️).
- **Types**: No type hints required (simple script), but return tuples documented in docstrings.
- **Config**: All settings via environment variables (see docker-compose.yml). No hardcoded values.
- **Logging**: Print statements with emoji prefixes for user visibility (✅, ❌, ⚠️, 🔍).
