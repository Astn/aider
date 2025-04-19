# Runner Configs for Aider Plugins

This document describes the structure and schema for the `runner-configs.json` file used in Aider plugins. The file centralizes configurations for various runners, making it easier to manage external processes, container runtimes, and script executions across different operating systems. Enhancements include optional install/health checks and support for dynamic placeholder substitution from plugin manifests, ensuring language-agnostic robustness.

## Purpose
The `runner-configs.json` file allows users to define reusable configurations for runners without cluttering individual plugin manifests. Each configuration specifies how to execute commands, handle volume mounts, perform health checks, and ensure compatibility with major OSes. Runners are designed to be language-agnostic, supporting any executable.

## Schema Overview
The file is a JSON object where each key is a unique configuration name (e.g., "docker"). Each value is an object with the following fields:

- **type**: String. The category of the runner (e.g., "Container Runtime", "Orchestration Tool", "Script Runner").
- **command_exists**: Optional string. A command to check if a resource (e.g., image) exists, such as "docker image inspect {image}".
- **command_install**: Optional string. A command to install or fetch a resource, such as "docker pull {image}".
- **command_run**: String. The command template for running the process, using placeholders like `{mount.host_path}`, `{mount.path}`, `{image}`, and `{command}`.
- **description**: String. A brief description of the runner, including any requirements and setup notes.
- **supported_host_os**: String. A description of supported operating systems and any specific setup notes (e.g., "Linux: Native, Windows: Native (via WSL2/VM), macOS: VM").

### Common Placeholders
The following placeholders are available in `command_exists`, `command_install`, and `command_run` for consistency:
- `{host_path}`: The path to the working directory on the host machine (deprecated in favor of mount-specific placeholders).
- `{container_path}`: The path inside the container where the host directory is mounted (use `mount.path` in manifests).
- `{image}`: The container image to use (can be referenced from plugin manifest).
- `{command}`: The actual command or process to execute.
- `{job_file}`: The path to a job file (for orchestration tools).
- `{pod}`: The pod name (for Kubernetes-based tools).
- `{allocation}`: The allocation ID (for Nomad).
- `{kubeconfig}`: The path to the kubeconfig file (for Kubernetes tools).
- `{shell}`: The shell to use (for script runners).
- **Dynamic Plugin Placeholders**: Prefix with `plugin.` followed by a JSON path (e.g., `{plugin.image}`, `{plugin.mount.host_path}`, or `{plugin.hooks[0]}`) to access values from the associated plugin's manifest.json. This allows for flexible, manifest-driven command construction.

### Key Considerations
- **Health and Install Checks**: Use `command_exists` to validate resource availability and `command_install` to fetch them if needed. These should be run before execution to ensure reliability.
- **Container Fetching**: The `command_install` field can be used for pulling images, improving reliability for container-based plugins.
- **Dynamic Placeholders**: When substituting placeholders, the system must load and parse the plugin's manifest.json to resolve `plugin.*` references using `jsonpath_ng`. This ensures that runner configs can adapt to different plugins without hardcoding values.
- **Robustness**: All commands should handle errors gracefully, with support for multiple OSes to ensure broad compatibility. Language agnosticism is maintained by treating all processes as executables.

## Example Structure
Here's an example of how the JSON file is structured with the current fields:

```json
{
  "docker": {
    "type": "Container Runtime",
    "command_exists": "docker image inspect {image}",
    "command_install": "docker pull {image}",
    "command_run": "docker run -v {mount.host_path}:{mount.path} {image} {command}",
    "description": "Docker-compatible container engine. Supports language-agnostic executables with volume mounts.",
    "supported_host_os": "Linux: Native, Windows: Native (via WSL2/VM), macOS: VM"
  }
}
