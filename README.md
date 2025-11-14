# Claude Skills

A collection of custom skills for Claude Code.

## Available Skills

- **obdb-editor** - Add and manage OBD-II extended PIDs in OBDb vehicle signalset JSON files by converting CSV formulas to proper mul/div/add format

## Installing Skills

### Option 1: Via Marketplace (Recommended)

1. In Claude Code, run:
   ```
   /plugin marketplace add https://github.com/OBDb/.claude-skills
   ```

2. Install the skills you want:
   ```
   /plugin
   ```
   Then select the skills from the marketplace to install.

### Option 2: Download from GitHub Actions

1. Go to the [Actions tab](../../actions/workflows/package-skills.yml) in this repository
2. Click on the most recent successful workflow run
3. Scroll down to the **Artifacts** section
4. Download the skill you want (each skill is packaged as a separate zip file)
5. Go to [claude.ai/settings/capabilities](https://claude.ai/settings/capabilities)
6. Click **Add Skill** and upload the downloaded zip file

### Option 3: Manual Installation

1. Clone this repository:
   ```bash
   git clone <repository-url>
   cd .claude-skills
   ```

2. Zip the skill folder contents (not the folder itself):
   ```bash
   cd obdb-editor
   zip -r ../obdb-editor.zip .
   cd ..
   ```

3. Go to [claude.ai/settings/capabilities](https://claude.ai/settings/capabilities)
4. Click **Add Skill** and upload the `obdb-editor.zip` file

## Skill Structure

Each skill must contain:
- `Skill.md` - The skill definition with YAML frontmatter and instructions

### YAML Frontmatter

The frontmatter at the top of `Skill.md` must contain:

```yaml
---
name: skill-name
description: A description of what the skill does
license: License information (optional)
---
```

**Required fields:**
- `name` - The name of the skill
- `description` - A description of what the skill does

**Optional fields:**
- `license` - License information for the skill

## Development

### Validation

All pull requests are automatically validated to ensure the YAML frontmatter contains only valid fields. The validation workflow checks that:
- Only allowed fields are present (`name`, `description`, `license`)
- Required fields are included (`name`, `description`)
- No additional properties are defined

### Packaging

When changes are pushed to the `main` branch, skills are automatically:
1. Packaged into individual zip files (named after each skill folder)
2. Uploaded as GitHub Actions artifacts for easy download

## Contributing

1. Create a new folder for your skill
2. Add a `Skill.md` file with proper YAML frontmatter
3. Submit a pull request
4. Wait for the validation workflow to pass
