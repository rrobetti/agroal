# Agroal XA Connection Pooling Documentation

This directory contains comprehensive documentation analyzing Agroal's XA connection pooling capabilities and how to use them in your applications.

## 📚 Documentation Index

### Start Here

**[SUMMARY.md](SUMMARY.md)** - Start here for executive summary and direct answers
- Quick overview of findings
- Direct answers to feasibility questions  
- Recommendations for implementation
- Links to detailed documentation

### Comprehensive Analysis

**[XA_POOLING_ANALYSIS.md](XA_POOLING_ANALYSIS.md)** - Complete architectural analysis (40+ pages)
- ✅ Executive summary with YES/NO answers
- 🏗️ Architecture overview with diagrams
- 🔍 How Agroal XA pooling works
- 📦 Core component descriptions
- 📖 Usage guide and examples
- 📋 Minimal class set for pooling
- ⚖️ Comparison with HikariCP
- 💡 Recommendations and conclusions

### Quick Reference

**[XA_POOLING_QUICK_REFERENCE.md](XA_POOLING_QUICK_REFERENCE.md)** - One-page quick reference
- 🚀 TL;DR and quick start
- ⚡ Essential code snippets
- 📊 Key concepts at a glance
- 🎯 Decision matrix
- 🛠️ Common configurations
- ⚠️ Common pitfalls

### Visual Diagrams

**[ARCHITECTURE_DIAGRAMS.md](ARCHITECTURE_DIAGRAMS.md)** - Mermaid architecture diagrams
- 🎨 System architecture
- 🔄 Connection creation flow
- 🔀 State machine
- 🧵 Thread interaction patterns
- 💱 XA transaction flow
- 📦 Memory layout
- 🏗️ Class hierarchy
- 📊 Comparison diagrams
- ⏱️ Housekeeping timeline

### Code Examples

**[INTEGRATION_EXAMPLES.md](INTEGRATION_EXAMPLES.md)** - Working integration examples
- 📦 Maven/Gradle setup
- 💻 Example 1: Standalone XA pool
- 🔀 Example 2: Database proxy with dual pools (HikariCP + Agroal)
- 🔄 Example 3: Narayana transaction manager integration
- 🎮 Example 4: Manual XA transaction management
- 🍃 Example 5: Spring Boot integration
- ⚙️ Production configurations
- 🧪 Testing examples

## 🎯 Quick Navigation

### By Use Case

**I want to understand if Agroal can work for me:**
→ Start with [SUMMARY.md](SUMMARY.md)

**I want to understand how Agroal works:**
→ Read [XA_POOLING_ANALYSIS.md](XA_POOLING_ANALYSIS.md)  
→ View [ARCHITECTURE_DIAGRAMS.md](ARCHITECTURE_DIAGRAMS.md)

**I want to implement Agroal XA pooling:**
→ Go to [INTEGRATION_EXAMPLES.md](INTEGRATION_EXAMPLES.md)  
→ Use [XA_POOLING_QUICK_REFERENCE.md](XA_POOLING_QUICK_REFERENCE.md) for lookups

**I need visual understanding:**
→ See [ARCHITECTURE_DIAGRAMS.md](ARCHITECTURE_DIAGRAMS.md)

**I'm in a hurry:**
→ Read [XA_POOLING_QUICK_REFERENCE.md](XA_POOLING_QUICK_REFERENCE.md)

### By Role

**👨‍💼 Decision Maker / Architect:**
1. Read [SUMMARY.md](SUMMARY.md) - Executive summary
2. Review "Comparison with HikariCP" section in [XA_POOLING_ANALYSIS.md](XA_POOLING_ANALYSIS.md)
3. Check "Decision Matrix" in [XA_POOLING_QUICK_REFERENCE.md](XA_POOLING_QUICK_REFERENCE.md)

**👨‍💻 Developer / Implementer:**
1. Skim [SUMMARY.md](SUMMARY.md) for context
2. Jump to [INTEGRATION_EXAMPLES.md](INTEGRATION_EXAMPLES.md)
3. Copy relevant example and adapt
4. Use [XA_POOLING_QUICK_REFERENCE.md](XA_POOLING_QUICK_REFERENCE.md) during coding

**🎓 Student / Learner:**
1. Start with [SUMMARY.md](SUMMARY.md)
2. Read full [XA_POOLING_ANALYSIS.md](XA_POOLING_ANALYSIS.md)
3. Study diagrams in [ARCHITECTURE_DIAGRAMS.md](ARCHITECTURE_DIAGRAMS.md)
4. Try examples from [INTEGRATION_EXAMPLES.md](INTEGRATION_EXAMPLES.md)

**🔍 Researcher / Analyst:**
1. Read [XA_POOLING_ANALYSIS.md](XA_POOLING_ANALYSIS.md) thoroughly
2. Study all diagrams in [ARCHITECTURE_DIAGRAMS.md](ARCHITECTURE_DIAGRAMS.md)
3. Review implementation details in code examples
4. Cross-reference with actual Agroal source code

## 📋 Key Questions Answered

All documents collectively answer these questions:

### Feasibility
- ✅ **Can I use Agroal for XA connection pooling in isolation?** → YES
- ✅ **Do I need a full application server?** → NO
- ✅ **Can I use it alongside HikariCP?** → YES (recommended for your case)
- ✅ **Does it work without a transaction manager?** → YES (but TM recommended for full XA)

### Architecture
- 📐 **How does Agroal pool XA connections?** → Unified handling via XAConnection wrapper
- 🔄 **What's the connection lifecycle?** → State machine (NEW → CHECKED_IN → CHECKED_OUT → FLUSH → DESTROYED)
- 🧵 **How are connections shared across threads?** → TransferQueue + ThreadLocal cache
- 💾 **How much code is involved?** → ~4,300 lines (core + config + utilities)

### Implementation
- 📦 **What dependencies do I need?** → agroal-api + agroal-pool (optionally agroal-narayana)
- ⚙️ **How do I configure it?** → See examples in INTEGRATION_EXAMPLES.md
- 🔌 **How do I integrate with my app?** → Multiple patterns shown in examples
- 🧪 **How do I test it?** → Unit test examples provided

### Design Decisions
- 🤔 **Should I copy the code?** → NO - use as library
- ⚖️ **Agroal vs HikariCP?** → Use both: HikariCP for non-XA, Agroal for XA
- 🔀 **Need transaction manager?** → Optional, but recommended for distributed transactions
- 🎯 **Best approach for database proxy?** → Hybrid with routing logic

## 🎨 Viewing Mermaid Diagrams

The documentation uses Mermaid for diagrams. View them with:

**GitHub**: Native support (diagrams render automatically)

**VS Code**: Install "Markdown Preview Mermaid Support" extension

**IntelliJ IDEA**: Install "Mermaid" plugin

**Online**: https://mermaid.live/ (paste diagram code)

**CLI**: `npm install -g @mermaid-js/mermaid-cli`

## 📊 Documentation Statistics

- **Total Documents**: 5 files
- **Total Content**: ~90,000 words
- **Code Examples**: 15+ complete examples
- **Diagrams**: 10+ architectural diagrams
- **Coverage**: Complete analysis of XA pooling

## 🔗 Related Resources

### Agroal Project
- **GitHub**: https://github.com/agroal/agroal
- **Main README**: [../README.md](../README.md)
- **Source Code**: `../agroal-pool/src/main/java/io/agroal/pool/`

### Transaction Managers
- **Narayana**: https://narayana.io/
- **Atomikos**: https://www.atomikos.com/

### JDBC & XA Specifications
- **JDBC Spec**: https://jcp.org/en/jsr/detail?id=221
- **JTA Spec**: https://jcp.org/en/jsr/detail?id=907
- **XA Protocol**: X/Open XA standard

## 📝 Document Metadata

**Created**: January 2026  
**Agroal Version Analyzed**: 3.0-SNAPSHOT  
**Primary Use Case**: Database proxy with XA connection pooling  
**Target Audience**: Developers, architects, students  

## 🤝 Contributing

To improve this documentation:

1. Identify gaps or unclear sections
2. Create an issue describing the improvement
3. Submit a PR with updated documentation
4. Ensure diagrams render correctly
5. Add examples if applicable

## 📜 License

This documentation is provided under the same Apache License 2.0 as the Agroal project.

---

**Need help?** Start with [SUMMARY.md](SUMMARY.md) or jump directly to your use case above.
