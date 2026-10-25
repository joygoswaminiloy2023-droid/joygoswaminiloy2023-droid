
name: Generate Contribution Graph

on:
  schedule:
    - cron: "15 0 * * *"
  workflow_dispatch:
  push:
    branches:
      - main

permissions:
  contents: write

jobs:
  generate:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Generate contribution graph
        uses: yoshi389111/github-profile-3d-contrib@latest
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          USERNAME: ${{ github.repository_owner }}

      - name: Publish graph
        run: |
          mkdir -p assets
          if [ -d profile-3d-contrib ]; then
            cp profile-3d-contrib/*.svg assets/
          fi
          if [ -f profile-3d-contrib/profile-green-animate.svg ]; then
            cp profile-3d-contrib/profile-green-animate.svg assets/contribution-graph.svg
          fi
          git add assets/
          git diff --cached --quiet || (
            git config user.name "github-actions[bot]"
            git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
            git commit -m "Update contribution graph"
            git push
          )
