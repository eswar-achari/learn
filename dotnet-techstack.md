Here's the updated script that outputs the **transitive dependencies in JSON format**:

---

### ✅ Python Script (JSON Output)

```python
import subprocess
import os
import sys
import json
import logging

logging.basicConfig(level=logging.INFO, format="%(levelname)s: %(message)s")


class DotNetDependencyExtractor:
    def __init__(self, project_path):
        self.project_path = os.path.abspath(project_path)

    def validate_project_file(self):
        if not os.path.exists(self.project_path):
            logging.error(f"Project file not found: {self.project_path}")
            return False
        if not self.project_path.endswith(".csproj"):
            logging.error("Please provide a valid .csproj file.")
            return False
        return True

    def extract_dependencies(self):
        try:
            command = [
                "dotnet", "list", self.project_path,
                "package", "--include-transitive"
            ]
            logging.info(f"Running command: {' '.join(command)}")
            result = subprocess.run(command, capture_output=True, text=True, check=True)
            return result.stdout
        except subprocess.CalledProcessError as e:
            logging.error("Failed to extract dependencies.")
            logging.error(e.stderr)
            return None

    def parse_dependencies(self, output):
        dependencies = []
        capture = False
        for line in output.splitlines():
            if line.startswith("Project") or line.startswith("Top-level") or "---" in line:
                if "---" in line:
                    capture = True
                continue

            if capture and "|" in line:
                parts = [part.strip() for part in line.split("|") if part.strip()]
                if len(parts) >= 3:
                    dependencies.append({
                        "name": parts[0],
                        "requested_version": parts[1],
                        "resolved_version": parts[2]
                    })
        return dependencies

    def run(self):
        if not self.validate_project_file():
            return []
        output = self.extract_dependencies()
        if output:
            return self.parse_dependencies(output)
        return []


if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("Usage: python extract_dotnet_deps.py <path-to-csproj>")
        sys.exit(1)

    project_file = sys.argv[1]
    extractor = DotNetDependencyExtractor(project_file)
    deps = extractor.run()

    print(json.dumps(deps, indent=4))
```

---

### 🧪 Example Command

```bash
python extract_dotnet_deps.py ./MyApp/MyApp.csproj
```

---

### 📄 Sample JSON Output

```json
[
    {
        "name": "Serilog",
        "requested_version": "2.12.0",
        "resolved_version": "2.12.0"
    },
    {
        "name": "Serilog.Sinks.Console",
        "requested_version": "4.0.0",
        "resolved_version": "4.0.0"
    },
    {
        "name": "Microsoft.Extensions.Logging",
        "requested_version": "6.0.0",
        "resolved_version": "6.0.0"
    }
]
```

---

Let me know if you'd like the output saved to a `.json` file or structured by top-level vs transitive dependencies.
