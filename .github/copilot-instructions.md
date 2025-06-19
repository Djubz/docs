This documentation repository consists mainly of content written in Markdown format. These files are converted into HTML for displaying on a website. Most Markdown files become a single article on the documentation site. Other files contain reusable content which is inserted into multiple articles. The repository also contains YAML files (e.g. for variable text), image files, JavaScript/TypeScript files, etc.

## Content guidelines

### Bullet lists

The bulleted points in a bullet list should always be denoted in Markdown using an asterisk, not a hyphen.

### Using variables

Within Markdown files, with the exception of the `title` field in the metadata at the start of a file, **always use the Liquid syntax variables rather than text** if a variable has been defined for that text. This ensures consistency and makes it easier to update product names globally.

**Important**: Variables must be used in all content, including reusable content, data files, and regular articles. The only exception is the `title` field in frontmatter metadata.

For example:

| Use this variable                                        | Don't use this text      | File where variable is defined   |
| -------------------------------------------------------- | ------------------------ | -------------------------------- |
| `{% data variables.product.github %}`                    | GitHub                   | data/variables/product.yml       |
| `{% data variables.product.prodname_ghe_server %}`       | GitHub Enterprise Server | data/variables/product.yml       |
| `{% data variables.product.prodname_copilot_short %}`    | Copilot                  | data/variables/product.yml       |
| `{% data variables.product.prodname_copilot %}`          | GitHub Copilot           | data/variables/product.yml       |
| `{% data variables.copilot.copilot_code-review_short %}` | Copilot code review      | data/variables/copilot.yml       |
| `{% data variables.enterprise.prodname_managed_user %}`  | managed user account     | data/variables/enterprise.yml    |
| `{% data variables.code-scanning.codeql_workflow %}`     | CodeQL analysis workflow | data/variables/code-scanning.yml |

There are many more variables. These are stored in various YAML files within the `data/variables` directory.

**How to find variables**: Check the `data/variables` directory for existing variables before writing hardcoded text. Common variable files include:

* `data/variables/product.yml` - Product names and variations
* `data/variables/copilot.yml` - Copilot-specific terms
* `data/variables/enterprise.yml` - Enterprise-specific terms
* `data/variables/code-scanning.yml` - Code scanning terms

### Reusable text

Reusables are long strings of reusable text, such as paragraphs or procedural lists, that are referenced in multiple content files. This makes it easier for us to maintain content and ensure that it is accurate across all files where the content is needed.

Each reusable lives in its own Markdown file. The path and filename of each reusable determines what its path will be in the data object. For example, a file named `/data/reusables/foo/bar.md` will be accessible as `{% data reusables.foo.bar %}` in articles.

Examples where you should create a reusable:

* You are documenting a new feature for a public preview. You need to create a note to display in all new articles about the new feature. Create a new reusable for the note and use it in all articles where it is needed.
* You are documenting billing for a new feature and need to briefly mention how the feature is billed and link to content about billing in several articles. Create a new reusable with the brief mention and a link to the content on billing. Aim to use the reusable in all places where you want to mention billing for the feature.

### Links to other articles

`[AUTOTITLE]` is the **only correct way** to specify the title of a linked article when that article is another page on the docs.github.com site.

You can replace the placeholder link text `[AUTOTITLE]` only when linking to an anchor in the same article or when linking to an anchor in another article and the actual article title would be confusing.

Never use the `{% link %}` Liquid tag for internal documentation links. The `[AUTOTITLE]` placeholder automatically pulls the correct title and ensures links remain valid when titles change.

Examplesname: 'Copilot Environment Setup'

# **What it does**: Sets up the environment for Copilot coding agent to test content and script changes.
# **Why we have it**: Ensures Copilot can validate content with linters and formatters before making changes.
# **Who does it impact**: Copilot coding agent and developers using repository custom instructions.

on:
  workflow_dispatch:
    inputs:
      check_content:
        description: 'Check content files with content linter'
        required: false
        default: true
        type: boolean
      check_scripts:
        description: 'Check TypeScript/JavaScript/SCSS files with prettier and linter'
        required: false
        default: true
        type: boolean
      paths:
        description: 'Specific file paths to check (space-separated), or leave empty for changed files'
        required: false
        type: string

permissions:
  contents: read

jobs:
  copilot-setup-steps:
    if: github.repository == 'github/docs-internal'
    runs-on: ${{ github.repository == 'github/docs-internal' && 'ubuntu-20.04-xl' || 'ubuntu-latest' }}

    steps:
      - name: Check out repo
        uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1

      - name: Set up Node and dependencies
        uses: ./.github/actions/node-npm-setup

      - name: Get changed files if no specific paths provided
        if: inputs.paths == ''
        id: changed_files
        uses: ./.github/actions/get-changed-files
        with:
          files: |
            content/**
            data/**
            src/**/*.{ts,tsx,js,mjs}
            **/*.scss

      - name: Set file paths for checking
        id: set_paths
        run: |
          if [ -n "${{ inputs.paths }}" ]; then
            echo "files_to_check=${{ inputs.paths }}" >> $GITHUB_OUTPUT
          else
            echo "files_to_check=${{ steps.changed_files.outputs.filtered_changed_files }}" >> $GITHUB_OUTPUT
          fi

      - name: Run content linter on content/data files
        if: inputs.check_content == true && (contains(steps.set_paths.outputs.files_to_check, 'content/') || contains(steps.set_paths.outputs.files_to_check, 'data/'))
        env:
          FILES_TO_CHECK: ${{ steps.set_paths.outputs.files_to_check }}
        run: |
          # Filter for content and data files only
          CONTENT_FILES=$(echo "$FILES_TO_CHECK" | tr ' ' '\n' | grep -E '^(content|data)/' | tr '\n' ' ' || true)
          if [ -n "$CONTENT_FILES" ]; then
            echo "Running content linter on: $CONTENT_FILES"
            npm run lint-content -- --paths $CONTENT_FILES
          else
            echo "No content or data files to check"
          fi

      - name: Run prettier check on script files
        if: inputs.check_scripts == true
        env:
          FILES_TO_CHECK: ${{ steps.set_paths.outputs.files_to_check }}
        run: |
          # Filter for TypeScript, JavaScript, and SCSS files
          SCRIPT_FILES=$(echo "$FILES_TO_CHECK" | tr ' ' '\n' | grep -E '\.(ts|tsx|js|mjs|scss)$' | tr '\n' ' ' || true)
          if [ -n "$SCRIPT_FILES" ]; then
            echo "Running prettier check on: $SCRIPT_FILES"
            npm run prettier-check -- $SCRIPT_FILES
          else
            echo "No script files to check with prettier"
          fi

      - name: Run ESLint on script files
        if: inputs.check_scripts == true
        env:
          FILES_TO_CHECK: ${{ steps.set_paths.outputs.files_to_check }}
        run: |
          # Filter for TypeScript and JavaScript files only (ESLint doesn't handle SCSS)
          SCRIPT_FILES=$(echo "$FILES_TO_CHECK" | tr ' ' '\n' | grep -E '\.(ts|tsx|js|mjs)$' | tr '\n' ' ' || true)
          if [ -n "$SCRIPT_FILES" ]; then
            echo "Running ESLint on: $SCRIPT_FILES"
            npx eslint $SCRIPT_FILES
          else
            echo "No JavaScript/TypeScript files to lint"
          fi

      - name: Run TypeScript compiler check
        if: inputs.check_scripts == true && (contains(steps.set_paths.outputs.files_to_check, '.ts') || contains(steps.set_paths.outputs.files_to_check, '.tsx'))
        run: |
          echo "Running TypeScript compiler check"
          npm run tsc

      - name: Environment setup summary
        run: |
          echo "✅ Copilot environment setup completed successfully!"
          echo ""
          echo "Available commands for content validation:"
          echo "- Content linting: npm run lint-content -- --paths <file-paths>"
          echo "- Prettier formatting: npm run prettier-check -- <file-paths>"
          echo "- ESLint: npm run lint"
          echo "- TypeScript check: npm run tsc"
          echo "- All tests: npm test"
          echo ""
          echo "For more guidance, see .github/copilot-instructions.md":

* ✅ Correct: `For more information, see [AUTOTITLE](/copilot/getting-started-with-github-copilot).`
* ❌ Incorrect: `For more information, see [Using GitHub Copilot](/copilot/getting-started-with-github-copilot).`
* ❌ Incorrect: `For more information, see {% link /copilot/getting-started-with-github-copilot %}.`

### Creating a pull request

When creating a pull request as a result of a request to do so in Copilot Chat, the first line of the PR description should **always** be the following (in italics):

`_This pull request was created as a result of the following prompt in Copilot Chat._`

Then, within a collapsed section, quote the original prompt from Copilot Chat:

```markdown
<details>
<summary>Original prompt - submitted by @GITHUB-USER-ID</summary>

> [Original prompt text here]

</details>
```

This helps reviewers understand the context and intent behind the automated changes.

## Development and testing guidelines

### Content changes

Before committing content changes, always:

1. **Use the content linter** to validate content: `npm run lint-content -- --paths <file-paths>`
2. **Check for proper variable usage** in your content
3. **Verify [AUTOTITLE] links** point to existing articles
4. **Run tests** on changed content: `npm run test -- src/content-render/tests/render-changed-and-deleted-files.js`

### Script and code changes

For TypeScript, JavaScript, and SCSS files:

1. **Run Prettier** to check formatting: `npm run prettier-check`
2. **Run the linter**: `npm run lint`
3. **Run TypeScript checks**: `npm run tsc`
4. **Run relevant tests**: `npm test`

### Environment setup

When testing changes in your development environment:

1. Install dependencies: `npm ci`
2. For content changes, ensure the content linter runs successfully
3. For script changes, ensure all formatting and linting checks pass
4. Always verify your changes don't break existing functionality
