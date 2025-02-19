Thank you for your interest in contributing to Steel Book! All contributions are welcome no matter how big or small.
This includes (but is not limited to) filing issues, adding documentation, fixing bugs, creating examples, and improving
the structure of the documentation.

## Choosing an issue

If you'd like to contribute, please claim an issue by commenting, forking, and opening a pull request, even if empty.
This allows the maintainers to track who is working on what issue as to not overlap work.

## Issue Guidelines

Please follow these guidelines:

Before making changes:

- Choose a branch name that describes the issue you're working on.

While working on documentation:

- Submit a draft PR as soon as possible.
- Keep changes focused on a single topic or section to maintain clarity.
- Use clear, concise language.
- Maintain consistency in formatting and style.
- Ensure headings, links, and code snippets are correctly formatted.
- If adding images, include descriptive alt text.
- If restructuring content, ensure navigation remains intuitive.

After making changes:

- If you've moved or added documentation files, check for broken links.
- Ensure that the structure of the documentation remains logical.
- Make sure all references and examples align with the latest project updates.

## Running the Documentation Locally

After making contributions, ensure the documentation builds and runs correctly by following these steps:

1. Install [mdBook](https://rust-lang.github.io/mdBook/):

    ```bash
    cargo install mdbook
    ```

2. Build the book:

    ```bash
    mdbook build
    ```

3. Serve the book locally and open it in your browser:

    ```bash
    mdbook serve --open
    ```

This allows you to verify that your changes render correctly and do not break the documentation before submitting your
pull request.

## Need Help?

If you need assistance while contributing:

- Review existing documentation for style and formatting guidelines.
- Check existing issues and discussions for similar contributions.
- Reach out to the maintainers via GitHub issues.

Thank you for helping improve the Steel Book documentation!