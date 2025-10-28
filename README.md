# World of Warcraft Addon Development - Complete Knowledge Base

## 📚 Overview
<!-- CLAUDE_SKIP_START -->

This comprehensive knowledge base contains **everything** you need to create, debug, analyze, and maintain World of Warcraft addons. It was created by analyzing the complete WoW 11.2.7 (The War Within) UI source code and extracting patterns, APIs, events, and best practices.

## ✨ What's Included
<!-- CLAUDE_SKIP_END -->

### Complete Documentation (12 Guides + Quick Start)
- ✅ **00_MASTER_PROMPT.md** - Master overview and entry point
- ✅ **01_API_Reference.md** - WoW API functions organized by category
- ✅ **02_Event_System.md** - Complete event system documentation
- ✅ **03_UI_Framework.md** - XML, frames, widgets, templates, mixins ⭐
- ✅ **04_Addon_Structure.md** - TOC files, file organization, load order
- ✅ **05_Patterns_And_Best_Practices.md** - Coding patterns, performance, best practices ⭐
- ✅ **06_Data_Persistence.md** - Saved variables, database management ⭐
- ✅ **07_Blizzard_UI_Examples.md** - Real working code examples ⭐
- ✅ **08_Community_Addon_Patterns.md** - Ace3, LibStub, community frameworks ⭐
- ✅ **09_Addon_Libraries_Guide.md** - Complete library reference (LibStub, Ace3, LibDataBroker, etc.) ⭐
- ✅ **10_Advanced_Techniques.md** - Production-level patterns (cross-client, performance, multi-addon) ⭐
- ✅ **11_API_Migration_Guide.md** - API version migration, compatibility wrappers, update strategies ⭐ NEW!
- ✅ **QUICK_START_GUIDE.md** - Get started in 5 minutes ⭐

### Reference Data
- **API Documentation** - 513 API files analyzed (`api_extracted/`)
- **Event Lists** - Complete event reference (`events_extracted/`)
- **File Organization** - Categorized lists of 281 Blizzard addons (`file_lists/`)

### Source Code Analysis
- **3,417** Blizzard UI source files analyzed
- **21,514** community addon files available for reference
- Real-world patterns extracted from official Blizzard code

## 🚀 Quick Start
<!-- CLAUDE_SKIP_START -->

### For Complete Beginners
1. Read **QUICK_START_GUIDE.md** (5-minute tutorial)
2. Read **00_MASTER_PROMPT.md** (overview)
3. Follow the "Your First Addon" tutorial
4. Start coding!

### For Experienced Developers
1. Check **00_MASTER_PROMPT.md** for structure
2. Jump to specific guides based on what you're building
3. Reference **07_Blizzard_UI_Examples.md** for code patterns
4. Use **05_Patterns_And_Best_Practices.md** for optimization

<!-- CLAUDE_SKIP_END -->
## 📖 Documentation Structure

<!-- CLAUDE_SKIP_START -->
### Core Concepts (Read First)
| File | Topics | When to Read |
|------|--------|--------------|
| **00_MASTER_PROMPT.md** | Overview, TOC basics, file organization | Start here |
| **QUICK_START_GUIDE.md** | 5-minute tutorial, quick reference | First-time developers |
| **04_Addon_Structure.md** | TOC files, load order, dependencies | Creating new addons |
| **02_Event_System.md** | Events, event handlers, registration | Event-driven programming |

### UI Development
| File | Topics | When to Read |
|------|--------|--------------|
| **03_UI_Framework.md** | XML, frames, templates, mixins, layouts | Building UI |
| **07_Blizzard_UI_Examples.md** | Action buttons, scrolling, tooltips | Need working examples |

### Code & Data
| File | Topics | When to Read |
|------|--------|--------------|
| **05_Patterns_And_Best_Practices.md** | Mixins, events, performance, patterns | Writing good code |
| **06_Data_Persistence.md** | Saved variables, databases, profiles | Saving settings |
| **08_Community_Addon_Patterns.md** | Ace3, LibStub, localization, profiles | Using frameworks |
| **09_Addon_Libraries_Guide.md** | LibStub, Ace3, LibDataBroker, all libraries | Using libraries |
| **10_Advanced_Techniques.md** | Cross-client, event bucketing, profiling, multi-addon | Production-level addons |
| **11_API_Migration_Guide.md** | Version upgrades, API changes, compatibility | Updating for new patches |

### API Reference
| File | Topics | When to Read |
|------|--------|--------------|
| **01_API_Reference.md** | C_* APIs, function categories | Looking up API calls |
| **api_extracted/** | Detailed API lists | Need specific functions |
| **events_extracted/** | Event lists and payloads | Need event details |
<!-- CLAUDE_SKIP_END -->

## 🎯 Common Use Cases
<!-- CLAUDE_SKIP_START -->

### I want to...

**Create my first addon**
→ Read: `QUICK_START_GUIDE.md` → `04_Addon_Structure.md`

**Build action buttons**
→ Read: `03_UI_Framework.md` → `07_Blizzard_UI_Examples.md` (Action Buttons section)

**Save addon settings**
→ Read: `06_Data_Persistence.md` → `08_Community_Addon_Patterns.md` (Profile Systems)

**Create a scrolling list**
→ Read: `03_UI_Framework.md` (ScrollBox) → `07_Blizzard_UI_Examples.md` (Scroll Frames)

**Handle events**
→ Read: `02_Event_System.md` → `05_Patterns_And_Best_Practices.md` (Event-Driven)

**Show tooltips**
→ Read: `07_Blizzard_UI_Examples.md` (Tooltips section)

**Use mixins**
→ Read: `05_Patterns_And_Best_Practices.md` (Mixin Patterns) → `03_UI_Framework.md`

**Optimize performance**
→ Read: `05_Patterns_And_Best_Practices.md` (Performance section)

**Use Ace3 framework**
→ Read: `09_Addon_Libraries_Guide.md` (Ace3 Library Suite) → `08_Community_Addon_Patterns.md`

**Add minimap icon**
→ Read: `09_Addon_Libraries_Guide.md` (LibDBIcon section)

**Use libraries (LibStub, LibDataBroker, etc.)**
→ Read: `09_Addon_Libraries_Guide.md` (comprehensive library guide)

**Support multiple WoW versions (Classic, Retail, etc.)**
→ Read: `10_Advanced_Techniques.md` (Cross-Client Compatibility)

**Optimize for production/large-scale addons**
→ Read: `10_Advanced_Techniques.md` (Performance, Event Bucketing, Multi-Addon)

**Update addon for new WoW patch/expansion**
→ Read: `11_API_Migration_Guide.md` (API changes, migration patterns, compatibility)

**Track quests**
→ Read: `07_Blizzard_UI_Examples.md` (Quest Tracking) → `01_API_Reference.md`

<!-- CLAUDE_SKIP_END -->
## 🔍 Finding Information
<!-- CLAUDE_SKIP_START -->

### By Topic

**UI Framework & Layout:**
- XML structure: `03_UI_Framework.md`
- Templates: `03_UI_Framework.md`
- Mixins: `05_Patterns_And_Best_Practices.md`
- ScrollBoxes: `03_UI_Framework.md` + `07_Blizzard_UI_Examples.md`
- Layout frames: `03_UI_Framework.md`

**Data Management:**
- Saved variables: `06_Data_Persistence.md`
- Profiles: `08_Community_Addon_Patterns.md`
- Database patterns: `06_Data_Persistence.md`
- Migrations: `06_Data_Persistence.md`

**Code Patterns:**
- Mixins: `05_Patterns_And_Best_Practices.md`
- Events: `02_Event_System.md` + `05_Patterns_And_Best_Practices.md`
- State management: `05_Patterns_And_Best_Practices.md`
- Performance: `05_Patterns_And_Best_Practices.md`

**API Usage:**
- API categories: `01_API_Reference.md`
- API wrappers: `05_Patterns_And_Best_Practices.md`
- C_* namespaces: `01_API_Reference.md`

**Community Patterns:**
- Ace3: `08_Community_Addon_Patterns.md`
- LibStub: `08_Community_Addon_Patterns.md`
- Localization: `08_Community_Addon_Patterns.md`
- Slash commands: `08_Community_Addon_Patterns.md`

### By File Type

**Looking at Blizzard source code?**
- Find addon categories: `file_lists/UI_Core_Systems.md`
- Most important addon: `Blizzard_SharedXML` (see `file_lists/UI_Core_Systems.md`)
- Action buttons: `Blizzard_ActionBar` (see `07_Blizzard_UI_Examples.md`)
- Buffs/debuffs: `Blizzard_BuffFrame` (see `07_Blizzard_UI_Examples.md`)

**Looking for API functions?**
- Browse by category: `01_API_Reference.md`
- Full API list: `api_extracted/00_API_INDEX.md`
- By category: `api_extracted/00_API_BY_CATEGORY.md`

**Looking for events?**
- Event system: `02_Event_System.md`
- Full event list: `events_extracted/00_EVENTS_INDEX.md`

## 📁 Directory Structure

```
WoW_Addon_Dev_Knowledge_Base/
│
├── README.md (this file)
├── QUICK_START_GUIDE.md
├── 00_MASTER_PROMPT.md
├── 01_API_Reference.md
├── 02_Event_System.md
├── 03_UI_Framework.md ⭐
├── 04_Addon_Structure.md
├── 05_Patterns_And_Best_Practices.md ⭐
├── 06_Data_Persistence.md ⭐
├── 07_Blizzard_UI_Examples.md ⭐
├── 08_Community_Addon_Patterns.md ⭐
├── 09_Addon_Libraries_Guide.md ⭐
├── 10_Advanced_Techniques.md ⭐
├── 11_API_Migration_Guide.md ⭐ NEW!
│
├── api_extracted/
│   ├── 00_API_INDEX.md
│   ├── 00_API_BY_CATEGORY.md
│   └── 00_API_STATISTICS.md
│
├── events_extracted/
│   └── 00_EVENTS_INDEX.md
│
└── file_lists/
    ├── README.md
    ├── 00_ALL_BLIZZARD_ADDONS.txt
    └── UI_Core_Systems.md
```
<!-- CLAUDE_SKIP_END -->

## 💡 Tips for Success

<!-- CLAUDE_SKIP_START -->
### Best Practices
1. ✅ **Start small** - Begin with a simple addon
2. ✅ **Study examples** - Read Blizzard's code in `07_Blizzard_UI_Examples.md`
3. ✅ **Use patterns** - Follow patterns in `05_Patterns_And_Best_Practices.md`
4. ✅ **Save properly** - Read `06_Data_Persistence.md` before saving data
5. ✅ **Test frequently** - Use `/reload` often during development

### Common Mistakes to Avoid
1. ❌ Don't pollute global namespace
2. ❌ Don't use OnUpdate for everything (use events/timers)
3. ❌ Don't forget to validate saved variables
4. ❌ Don't create frames every time (use pools)
5. ❌ Don't hardcode strings (use localization)

### Debugging
```lua
-- Enable errors
/console scriptErrors 1

-- Print to chat
print("Debug:", value)

-- Dump table
/dump MyTable

-- Show frames
/fstack

-- Trace events
/eventtrace
```

## 🛠️ Development Tools

### In-Game
- `/reload` - Reload UI
- `/fstack` - Show frame stack
- `/eventtrace` - Monitor events
- `/dump` - Print variables
- `/console scriptErrors 1` - Show Lua errors

### Addons
- **BugSack** - Error capture
- **DevTool** - Variable inspector
- **Blizzard_DebugTools** - Built-in debug tools

<!-- CLAUDE_SKIP_END -->
## 📍 File Paths

<!-- CLAUDE_SKIP_START -->
### WoW UI Source (for reference)
```
D:\Games\World of Warcraft\_retail_\Interface\+wow-ui-source+ (11.2.7)\
```

### Your Addons
```
D:\Games\World of Warcraft\_retail_\Interface\AddOns\
```

### Saved Variables
```
WTF\Account\[Account]\SavedVariables\
WTF\Account\[Account]\[Server]\[Character]\SavedVariables\
```
<!-- CLAUDE_SKIP_END -->

## 🎓 Learning Path
<!-- CLAUDE_SKIP_START -->

### Week 1: Basics
- [ ] Read `QUICK_START_GUIDE.md`
- [ ] Create your first addon (5-minute tutorial)
- [ ] Read `00_MASTER_PROMPT.md`
- [ ] Read `04_Addon_Structure.md`
- [ ] Experiment with slash commands

### Week 2: UI & Events
- [ ] Read `02_Event_System.md`
- [ ] Read `03_UI_Framework.md`
- [ ] Create a simple frame
- [ ] Handle PLAYER_LOGIN event
- [ ] Read `07_Blizzard_UI_Examples.md`

### Week 3: Data & Patterns
- [ ] Read `06_Data_Persistence.md`
- [ ] Implement saved variables
- [ ] Read `05_Patterns_And_Best_Practices.md`
- [ ] Refactor code using mixins
- [ ] Optimize performance

### Week 4: Advanced
- [ ] Read `08_Community_Addon_Patterns.md`
- [ ] Read `09_Addon_Libraries_Guide.md`
- [ ] Try Ace3 framework
- [ ] Create scrolling lists
- [ ] Implement profiles
- [ ] Study Blizzard source code

### Week 5+: Production-Level
- [ ] Read `10_Advanced_Techniques.md`
- [ ] Read `11_API_Migration_Guide.md`
- [ ] Implement cross-client compatibility
- [ ] Add performance profiling
- [ ] Optimize with event bucketing
- [ ] Build multi-addon architecture
- [ ] Learn to migrate addons for new patches

<!-- CLAUDE_SKIP_END -->
## 📚 Additional Resources

<!-- CLAUDE_SKIP_START -->
### Official Blizzard Code
All examples in this knowledge base reference:
- WoW Version: **11.2.7 (The War Within)**
- Files Analyzed: **3,417** Blizzard UI files
- Addons Documented: **281** official addons

### External Links
- **Wowpedia**: https://wowpedia.fandom.com/wiki/World_of_Warcraft_API
- **WoW AddOn Discord**: Community support
- **CurseForge**: Download popular addons for study
- **Wago.io**: Addon hosting and sharing

<!-- CLAUDE_SKIP_END -->
## ✅ Completeness Checklist

<!-- CLAUDE_SKIP_START -->
This knowledge base includes:
- ✅ Complete API reference (513 files analyzed)
- ✅ Complete event system documentation
- ✅ Complete UI framework guide
- ✅ Complete coding patterns and best practices
- ✅ Complete data persistence guide
- ✅ Real working code examples
- ✅ Community framework patterns
- ✅ Quick start tutorial
- ✅ File organization and categorization
- ✅ Blizzard source code analysis

<!-- CLAUDE_SKIP_END -->
## 🎯 Next Steps

<!-- CLAUDE_SKIP_START -->
1. **New Developer?** → Start with `QUICK_START_GUIDE.md`
2. **Need specific feature?** → Use the "I want to..." section above
3. **Want deep knowledge?** → Read all guides in order (00-08)
4. **Ready to code?** → Reference guides as needed while building

---

## 📝 Version Information

**Knowledge Base Version:** 1.1
**WoW Version:** 11.2.7 (The War Within)
**Last Updated:** 2025-10-19
**Files Analyzed:** 3,417 Blizzard UI files + 21,514 community addon files
**Documentation Files:** 12 comprehensive guides

---

<!-- CLAUDE_SKIP_END -->
**Happy Addon Development!** 🚀

For questions about using this knowledge base, refer to `00_MASTER_PROMPT.md` or `QUICK_START_GUIDE.md`.
