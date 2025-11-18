# Architecture Diagrams

This directory contains architecture diagrams for the Elsa Workflows distributed system.

## Available Diagrams

### C4 Container Diagram
**File**: `c4-container-diagram.puml`

**Description**: Shows the distributed architecture of Elsa Workflows with all major components including API Gateway, Workflow Server, Worker Nodes, PostgreSQL databases, AWS SQS message queue, and Redis cache.

**Technologies**:
- PlantUML with C4 extension
- Format: `.puml`

**How to Render**:

#### Option 1: VS Code (Recommended)
1. Install the PlantUML extension
2. Open `c4-container-diagram.puml`
3. Press `Alt+D` to preview

#### Option 2: Online
1. Visit [PlantUML Web Server](http://www.plantuml.com/plantuml/uml/)
2. Copy and paste the diagram content
3. Click "Submit" to render

#### Option 3: Local PlantUML
```bash
# Install PlantUML
brew install plantuml  # macOS
# or download from https://plantuml.com/download

# Generate PNG
plantuml c4-container-diagram.puml

# Generate SVG (recommended for documentation)
plantuml -tsvg c4-container-diagram.puml
```

#### Option 4: Docker
```bash
docker run --rm -v $(pwd):/data plantuml/plantuml:latest -tsvg /data/c4-container-diagram.puml
```

## Diagram Components

The C4 Container Diagram includes:

### User Personas
- **Workflow Developer**: Creates and manages workflows
- **System Administrator**: Manages infrastructure
- **External User**: Triggers workflows via API

### Containers
1. **API Gateway**: REST API entry point
2. **Workflow Server**: Hosts workflow runtime
3. **Workflow Runtime**: Core execution engine
4. **Worker Nodes**: Horizontally scalable task processors
5. **Workflow Studio**: Visual designer (Blazor WebAssembly)
6. **Workflow Scheduler**: Time-based trigger management

### Data Stores
1. **Workflow Database** (PostgreSQL): Main persistence layer
2. **Runtime State Store** (PostgreSQL): Execution state and locks
3. **Message Queue** (AWS SQS): Task distribution
4. **Distributed Cache** (Redis): Performance optimization

### External Systems
- External HTTP Services
- Email Service (SMTP)
- Monitoring & Logging (Application Insights/CloudWatch/Datadog)

## C4 Model Context

This diagram follows the [C4 Model](https://c4model.com/) for visualizing software architecture:

- **Level 1 - System Context**: Shows how the system fits into the wider environment
- **Level 2 - Container**: Shows the high-level technology choices and responsibilities (this diagram)
- **Level 3 - Component**: Shows components within a container
- **Level 4 - Code**: Shows implementation details

## Future Diagrams

Planned additions:
- Component diagram for Workflow Runtime Engine
- Deployment diagram for AWS infrastructure
- Sequence diagrams for key workflows:
  - Workflow creation and publication
  - Synchronous workflow execution
  - Asynchronous workflow execution via queue
  - Scheduled workflow execution
- Data flow diagrams
- Security architecture diagram

## Tools and Resources

### PlantUML Resources
- [PlantUML Documentation](https://plantuml.com/)
- [C4-PlantUML](https://github.com/plantuml-stdlib/C4-PlantUML)
- [C4 Model Website](https://c4model.com/)

### VS Code Extensions
- **PlantUML**: (jebbs.plantuml) - Preview and export PlantUML diagrams
- **Markdown Preview Enhanced**: (shd101wyy.markdown-preview-enhanced) - Enhanced markdown with diagram support

### Alternative Diagram Tools
- [Draw.io](https://app.diagrams.net/) - Visual diagram editor
- [Mermaid](https://mermaid.js.org/) - Markdown-like diagram syntax
- [Structurizr](https://structurizr.com/) - C4 model tooling

## Contributing

When adding new diagrams:
1. Use PlantUML format for consistency
2. Follow C4 model conventions
3. Include clear descriptions and legends
4. Add entry to this README
5. Export to SVG for version control
6. Keep source `.puml` files in this directory

## References
- [Main Architecture Documentation](../README.md)
- [Component Documentation](../components/)
