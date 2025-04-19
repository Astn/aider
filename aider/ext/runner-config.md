# Runner Configs for Aider Plugins

This document describes the structure and schema for the `runner-configs.json` file used in Aider plugins. The file centralizes configurations for various runners, making it easier to manage external processes, container runtimes, and script executions across different operating systems. Enhancements include optional install/health checks and support for dynamic placeholder substitution from plugin manifests, ensuring language-agnostic robustness.

## Purpose
The `runner-configs.json` file allows users to define reusable configurations for runners without cluttering individual plugin manifests. Each configuration specifies how to execute commands, handle volume mounts, perform health checks, and ensure compatibility with major OSes. Runners are designed to be language-agnostic, supporting any executable.

## Schema Overview
The file is a JSON object where each key is a unique configuration name (e.g., "docker-config"). Each value is an object with the following fields:

- **type**: String. The category of the runner (e.g., "Container Runtime", "Orchestration Tool", "Script Runner").
- **command_template**: String. A template string for running commands, using placeholders like `{host_path}`, `{container_path}`, `{image}`, `{command}`, and dynamic plugin references like `{plugin.foo.bar}` (e.g., to access values from the plugin's manifest.json).
- **description**: String. A brief description of the runner, including any requirements, health check details, and setup notes.
- **supported_host_os**: String. A description of supported operating systems and any specific setup notes (e.g., "Linux: Native, Windows: Native (via WSL2/VM), macOS: VM").
- **health_check**: Optional string or array of strings. Commands to run for verifying the runner is installed and functional (e.g., "docker --version").
- **install_command**: Optional object with OS-specific install instructions (e.g., {"linux": "sudo apt-get install -y docker.io", "windows": "choco install docker-desktop", "macos": "brew install docker"}). These are informational and may require user confirmation before execution.
- **fetch_command**: Optional string. A template for fetching resources, such as pulling container images (e.g., "docker pull {image}"). Supports placeholders for dynamic substitution.

### Common Placeholders
The following placeholders are available in `command_template`, `health_check`, `install_command`, and `fetch_command` for consistency:
- `{host_path}`: The path to the working directory on the host machine.
- `{container_path}`: The path inside the container where the host directory is mounted.
- `{image}`: The container image to use (can be referenced from plugin manifest).
- `{command}`: The actual command or process to execute.
- `{job_file}`: The path to a job file (for orchestration tools).
- `{pod}`: The pod name (for Kubernetes-based tools).
- `{allocation}`: The allocation ID (for Nomad).
- `{kubeconfig}`: The path to the kubeconfig file (for Kubernetes tools).
- `{shell}`: The shell to use (for script runners).
- **Dynamic Plugin Placeholders**: Prefix with `plugin.` followed by a JSON path (e.g., `{plugin.external_process}`, `{plugin.container_image}`, or `{plugin.hooks[0]}`) to access values from the associated plugin's manifest.json. This allows for flexible, manifest-driven command construction.

### Key Considerations
- **Health and Install Checks**: Use `health_check` to validate runner availability (e.g., check exit codes). `install_command` provides OS-targeted suggestions but should not be executed automatically without user consent to avoid security risks.
- **Container Fetching**: The `fetch_command` field automates pulling images or other resources, improving reliability for container-based plugins.
- **Dynamic Placeholders**: When substituting placeholders, the system must load and parse the plugin's manifest.json to resolve `plugin.*` references. This ensures that runner configs can adapt to different plugins without hardcoding values.
- **Robustness**: All commands should handle errors gracefully, with support for multiple OSes to ensure broad compatibility. Language agnosticism is maintained by treating all processes as executables.

## Example Structure
Here's an example of how the JSON file is structured with the new fields:

```json
{
  "docker-config": {
    "type": "Container Runtime",
    "command_template": "docker run -v {host_path}:{container_path} {image} {command}",
    "health_check": "docker --version",
    "install_command": {
      "linux": "sudo apt-get install -y docker.io",
      "windows": "choco install docker-desktop",
      "macos": "brew install docker"
    },
    "fetch_command": "docker pull {image}",
    "description": "Docker-compatible container engine. Supports language-agnostic executables with volume mounts.",
    "supported_host_os": "Linux: Native, Windows: Native (via WSL2/VM), macOS: VM"
  }
}
```
