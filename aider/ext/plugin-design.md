# Aider Plugin Support Epic

## Overview
This epic outlines the design and implementation plan for adding plugin support to the Aider project. Plugins will enable extensible features, such as integrating with external processes (e.g., similar to VSCode's Language Server Protocol). The goal is to make Aider more modular, allowing users to add capabilities like code analysis or remote services without core modifications. Importantly, plugins are designed to be language-agnostic, meaning they can be written in any language or as simple executables, as long as they can be executed via the specified runners (e.g., native or container-based). No specific runtime, such as Python, is required for the plugins themselves.

Key Objectives:
- Activate plugins on demand with user confirmation.
- Ensure reliable, language-agnostic communication with external processes, supporting configurable container runtimes and improved runner health checks.
- Maintain security, usability, and compatibility with existing workflows across major operating systems, including WSL2 on Windows and native support.
- Make the system robust by adding validation, error handling, fallback mechanisms, and dynamic interactions between runner configs and plugin manifests.

## Key Concepts and Elaborations

### 1. Plugin Activation on Demand with User Confirmation
Plugins will be activated via explicit user commands (e.g., `/plugin activate <name>` in `aider\commands.py`), mirroring the confirmation flow for adding files in `aider\io.py`. This design prioritizes user control and includes preliminary checks for runner availability.

Elaboration Ideas:
- **Command Integration**: Extend `aider\commands.py` to include health and install checks before activation. For example, run the runner's `health_check` command and prompt the user if issues are found (e.g., "Runner 'docker' is not installed. Would you like to run the install command?").
- **User Experience**: Display a summary of the plugin's capabilities and any required runner setups. Use `aider\io.py` for confirmations, including timeouts for responsiveness.
- **Potential Enhancement**: Integrate OS-specific logic to handle diverse environments, ensuring that health checks and installs are targeted.

### 2. Communication Check and Process Launch
After activation, Aider will perform runner-specific checks (e.g., health and fetch), then attempt communication via stdIO. If needed, launch the process based on manifest and runner details. For container-based runners, include fetching images to ensure resources are available. Plugins are language-agnostic, with dynamic placeholder substitution allowing runner configs to reference manifest values using `jsonpath_ng` for JSON path queries.

Elaboration Ideas:
- **Communication Mechanism**: Use stdIO checks in `aider\run_cmd.py`. Before launching, execute `health_check` and `fetch_command` from the runner config. For example:
  - Check runner health (e.g., "docker --version" should return a valid version).
  - Fetch resources if applicable (e.g., "docker pull {image}", with `{image}` substituted from the manifest).
  - On Windows, detect WSL2 for compatibility; fall back to native or error out.
  - **Dynamic Placeholder Substitution with jsonpath_ng**: When constructing commands, use the `jsonpath_ng` library to query the plugin's manifest.json. For instance, for a placeholder like `{plugin.container_image}`, `jsonpath_ng` parses the JSON path and retrieves the value. This is implemented in `aider\run_cmd.py` by loading the manifest, parsing the placeholder, and substituting it into the command template. Example in code:
    - Load manifest: `manifest_dict = json.loads(manifest_content)`
    - Parse JSON path: `jsonpath_expr = jsonpath_ng.parse('container_image')`
    - Extract value: `value = jsonpath_expr.find(manifest_dict)[0].value`
    - Substitute: `substituted_command = command_template.format(**substitution_dict)`
  - This approach ensures efficient and secure access to nested manifest fields without manual parsing.
- **Manifest-Driven Configuration**: Each plugin's manifest includes fields like:
  - `external_process`: Path to the executable.
  - `runner`: "native" or "container".
  - `runner_config`: Reference to a runner in `runner-configs.json`.
  - `container_image`: Image name for container runners.
  - `container_mounts`: Volume mount details.
  - `auto_discovery`: Commands for capability discovery.
  - Dynamic references in runner configs (e.g., `{plugin.container_image}`) allow for flexible command construction without duplication.
- **Error Handling and Fallbacks**: If health checks fail, suggest running `install_command` or deactivate the plugin. Implement retries for fetch operations and validate all placeholders during substitution. Handle JSON path errors gracefully, such as logging warnings if a path is invalid.
- **Process Management**: Use `aider\watch.py` for monitoring, ensuring processes can be stopped and resources cleaned up.

### 3. AI Invocation of Plugins
Plugins are not only invoked by user commands but can also be triggered or suggested by the AI (LLM) itself during interactions. This section discusses how all plugins (active and inactive) are presented to the AI, the required response template for invocation or activation suggestions, and how the AI can proactively engage with the plugin system.

Elaboration Ideas:
- **Presentation to the AI**: When sending prompts to the LLM, include a comprehensive description of all discovered plugins, regardless of their activation status. For example, append a section to the system prompt like: "Available plugins: - grep-plugin (inactive): Provides code search capabilities via the 'on_code_search' hook. - cowsay-plugin (active): Decorates messages with ASCII art via the 'on_message_decorate' hook. Inactive plugins can be activated using the /activate command. To invoke an active plugin, use /invoke plugin_name arguments. If a plugin is inactive and relevant, you can suggest activation with /suggest_activate plugin_name." This allows the AI to see the full set of tools, understand their services, and decide whether to invoke them directly or recommend activation based on the context, enhancing the conversational flow.
- **Response Template for Invocation and Suggestions**: The AI must use standardized response formats to interact with plugins:
  - For invoking an active plugin: Respond with `/invoke plugin_name arguments`, e.g., "/invoke grep-plugin pattern='error' file='code.py'".
  - For suggesting activation of an inactive plugin: Respond with `/suggest_activate plugin_name`, e.g., "/suggest_activate grep-plugin" to prompt the user for confirmation. Aider's code will parse these responses (in `aider\coders\base_coder.py` or `aider\commands.py`) and handle them accordingly—e.g., executing the plugin for `/invoke` or showing a confirmation prompt for `/suggest_activate`. This template ensures consistency, with validation to handle errors like unknown plugins or malformed commands.
- **Benefits**: By exposing inactive plugins, the AI can provide more intelligent suggestions, such as recommending a code search tool when the user describes a problem. This integration maintains security by requiring user confirmation for activation while leveraging the AI's capabilities for better assistance. It also supports Aider's modularity, as the AI can adapt to new plugins without code changes.
- **Potential Enhancement**: Include plugin descriptions from the manifest in the AI prompt for richer context, using dynamic generation to keep the information up-to-date.

### 4. Example Use Case: Running .NET Tests with Container Runner
This section provides a real-world example to illustrate the plugin system's capabilities. Consider a scenario where a developer is using Aider to modify C# code, and the AI can invoke a plugin to run `dotnet test` in a containerized environment, with results fed back to the AI for analysis.

Elaboration Ideas:
- **Use Case Description**: A plugin named "dotnet-test-plugin" is created to run .NET tests. When the AI makes code changes, it can invoke the plugin to execute `dotnet test` using a container runner (e.g., Docker or Podman). The working directory is mounted into the container, allowing the plugin to access the latest code. After the test run, the output (e.g., test pass/fail results) is captured and returned to the AI, which can then interpret the results, suggest fixes, or provide feedback in the conversation.
- **Plugin Manifest Configuration**: The manifest.json for this plugin would specify:
  - `external_process`: "dotnet test"
  - `runner`: "container"
  - `runner_config`: "docker-config" or "podman-config"
  - `container_image`: "mcr.microsoft.com/dotnet/sdk:6.0" (or a suitable .NET SDK image)
  - `container_mounts`: {"host_path": "{working_dir}", "container_path": "/app"} to mount the working directory.
  - `hooks`: ["on_code_change"] or similar, allowing the AI to trigger it after modifications.
  - `auto_discovery`: Commands to check for test frameworks or capabilities.
- **Runner Integration**: Use the `docker-config` or `podman-config` from runner-configs.json, with dynamic placeholders like `{plugin.container_image}` for the image and `{command}` for "dotnet test". The `fetch_command` ensures the container image is pulled if needed.
- **AI Interaction**: In the AI prompt, the plugin is listed (e.g., "dotnet-test-plugin (inactive): Runs .NET tests in a containerized environment."). The AI can use `/invoke dotnet-test-plugin` after code changes, and Aider captures the output via stdIO, passing it back to the AI for response generation. For instance, if tests fail, the AI could say, "Tests failed with errors; here's a suggestion to fix it."
- **Benefits**: This use case demonstrates real-world applicability, such as in CI/CD workflows, where the AI can automate testing and debugging, improving productivity. It highlights the system's flexibility with containerization for isolated, reproducible environments.
- **Potential Challenges and Mitigations**: Ensure the mounted directory handles file permissions correctly; mitigate by using read-only mounts if needed. Handle test output parsing to avoid overwhelming the AI, perhaps by summarizing results in Aider's code before sending to the LLM.

## Design Approach
### High-Level Architecture
- **Plugin Directory Structure**: 
  - `aider/ext/plugins/`: Root for plugins, with `manifest.json` containing all plugin-specific data, including descriptions for AI presentation.
  - `aider/ext/runner-configs.json`: Central runner configs with enhanced fields for checks and dynamic placeholders.
- **Lifecycle Flow**:
  1. **Discovery**: Scan for plugins on startup in `aider\coders\base_coder.py` and make all plugins available for AI context.
  2. **Activation**: Confirm with user, run runner health checks, and fetch resources if needed.
  3. **Communication**: Attempt stdIO connection with dynamic command substitution using `jsonpath_ng`.
  4. **Execution**: Hook into core methods, support AI-driven invocations and suggestions, using manifest values in runner templates.
  5. **Deactivation/Cleanup**: Stop processes and handle any fetched resources.

- **Integration Points**:
  - **aider\coders\base_coder.py**: Load plugins and modify AI prompts to include all plugins with status, handle invocation parsing.
  - **aider\io.py**: Handle user prompts for health/install issues and activation confirmations triggered by AI suggestions.
  - **aider\commands.py**: Manage activation commands and parse AI responses for `/invoke` and `/suggest_activate`.
  - **aider\run_cmd.py**: Execute commands with placeholder substitution, including `jsonpath_ng` for manifest parsing.
  - **aider\prompts.py**: Custom prompts for AI context updates and error handling.

### Risks and Mitigations
- **Security Risks**: Arbitrary code execution in checks or installs. Mitigation: Run checks in a sandboxed way, require user confirmation, and validate inputs.
- **Performance Overhead**: Including all plugins in AI prompts could increase token usage. Mitigation: Make the plugin list concise, cache it, and use it only when relevant (e.g., in specific chat modes).
- **Compatibility**: OS-specific commands might not work everywhere. Mitigation: Use descriptive fields and fallback to user prompts.
- **Dynamic Placeholder Handling**: Parsing JSON paths with `jsonpath_ng` could introduce errors if paths are malformed. Mitigation: Add input validation and handle exceptions during substitution.
- **AI Invocation Risks**: The AI might generate invalid or misleading suggestions for inactive plugins. Mitigation: Implement strict parsing, limit suggestions to discovered plugins, and always require user confirmation for activation to prevent unauthorized actions.

### Next Steps
1. Update `runner-configs.json` examples to include the new fields.
2. Implement placeholder substitution with `jsonpath_ng` in `aider\run_cmd.py`.
3. Add AI prompt modification for all plugins and parsing for `/suggest_activate` in `aider\coders\base_coder.py` and `aider\commands.py`.
4. Test the system with various plugins and OSes for robustness, including AI interactions and the new example use case.

This updated design enhances interaction between components, making the plugin system more reliable and flexible.

aider\ext\plugins\dotnet-test\manifest.json
