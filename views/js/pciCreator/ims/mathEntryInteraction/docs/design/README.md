# Math Entry Interaction PCI - Design Documentation

## Overview

This directory contains comprehensive design documentation for the Math Entry Interaction Portable Custom Interaction (PCI).

## Document Structure

### Japanese Documentation (ja/)

1. **01_概要設計書.md** (Overview Design Document)
   - System overview and purpose
   - Technology stack
   - Main features
   - System architecture
   - Non-functional requirements
   - Security considerations

2. **02_アーキテクチャ設計書.md** (Architecture Design Document)
   - Architecture style and patterns
   - Component structure
   - Data flow architecture
   - Event-driven architecture
   - Module dependencies
   - Design patterns used
   - Performance optimization strategies

3. **03_Runtimeモジュール仕様書.md** (Runtime Module Specification)
   - Runtime module file structure
   - Main file specifications (mathEntryInteraction.js)
   - responsesManagerFactory specification
   - mathEntryInteractionFactory detailed specification
   - MathQuill management methods
   - Gap mode methods
   - Toolbar methods
   - Response management methods
   - Helper modules (ambiguousSymbols, mathInPrompt)

4. **04_Creatorモジュール仕様書.md** (Creator Module Specification)
   - Creator module file structure
   - Widget.js specification
   - State management (Question, Correct, Map states)
   - Property form management
   - Tool definition and management
   - Alternative answer management
   - Score management
   - Event handlers

5. **05_インターフェース仕様書.md** (Interface Specification)
   - IMS PCI standard interface
   - PCI custom events
   - PCI configuration properties
   - TAO Creator interface
   - MathQuill API
   - Data formats

## Version Information

- **Version**: 2.8.0
- **Date**: 2026-01-11
- **PCI Identifier**: mathEntryInteraction

## Target Audience

- Developers extending or maintaining the PCI
- Technical architects reviewing the system design
- QA engineers understanding the system behavior
- Integration developers working with TAO platform

## Prerequisites

To understand these documents, familiarity with the following is recommended:

- JavaScript (ES6+)
- AMD module system
- IMS QTI specification
- TAO Assessment Platform
- MathQuill library
- Design patterns (Factory, State, Observer)

## Key Concepts

### Runtime vs Creator Modes

- **Runtime Mode**: Used when test takers are taking the assessment
- **Creator Mode**: Used when item authors are creating/editing items

### Gap Expression Mode

A special mode where the math expression contains editable "gaps" (placeholders) that test takers fill in.

### State Pattern

The Creator mode uses a State Pattern with three main states:
- **Question**: Editing the question and properties
- **Correct**: Transitional state
- **Map**: Defining correct answers and scoring

## Related Resources

- [IMS PCI Specification v1.0](http://www.imsglobal.org/assessment/pciv1p0cf/imsPCIv1p0cf.html)
- [MathQuill Documentation](http://docs.mathquill.com/)
- [TAO Platform Documentation](https://www.taotesting.com/)

## Contributing

When updating the design documents:

1. Keep all documents in sync
2. Update the version number and date
3. Add change history entries
4. Maintain the established format and structure

## License

GPL-2.0-only

---

*This documentation set was created to provide comprehensive technical reference for the Math Entry Interaction PCI.*
