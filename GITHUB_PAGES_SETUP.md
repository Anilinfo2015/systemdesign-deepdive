# GitHub Pages Setup

This repository is configured to automatically publish documentation to GitHub Pages.

## How it Works

The GitHub Actions workflow (`.github/workflows/pages.yml`) automatically:
1. Discovers all markdown files in the repository
2. Copies them to a `docs` directory
3. Includes the `index.html` landing page
4. Deploys everything to GitHub Pages

## Enabling GitHub Pages

To enable GitHub Pages for this repository:

1. Go to your repository on GitHub
2. Click on **Settings**
3. Navigate to **Pages** (in the left sidebar under "Code and automation")
4. Under **Source**, select **GitHub Actions**
5. Save your changes

Once enabled, the workflow will automatically run on every push to the `main` branch, and your site will be published.

## Accessing Your Site

After deployment, your site will be available at:
```
https://<username>.github.io/<repository-name>/
```

For this repository:
```
https://Anilinfo2015.github.io/systemdesign-deepdive/
```

## Manual Deployment

You can manually trigger a deployment by:
1. Going to the **Actions** tab
2. Selecting the "Deploy to GitHub Pages" workflow
3. Clicking **Run workflow**

## Adding New Content

Simply add new markdown files to any folder in the repository, and they will be automatically published when pushed to the `main` branch.

To add a new category:
1. Create a new folder (e.g., `database-patterns/`)
2. Add markdown files to that folder
3. Update the `index.html` file to include the new category in the navigation

## Local Testing

To test the site locally, you can use a simple HTTP server:

```bash
# Using Python
python -m http.server 8000

# Using Node.js
npx http-server
```

Then open your browser to `http://localhost:8000`
