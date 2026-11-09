# Glossary

The Structural terms of this architecture. Each means only what it says here, and a server doesn't reuse it for anything else.

| Term | Name in code | Meaning |
| --- | --- | --- |
| Feature | `<Feature>Feature` | One domain, and the unit of deletion |
| Capability | `Capabilities/<Capability>/` | One read or write surface, or one workflow |
| Descriptor | `<Capability>ToolDescriptor` | Declares a capability's tools to the server |
| Handler | `<Capability>ToolHandler` | Decodes arguments, calls its protocol, and encodes the result |
| Backend | `<Feature>Backend` | The feature's one implementation of its handlers' protocols |
| Tool error | `<Feature>ToolError` | The errors a feature's tools report, translated in the backend |
