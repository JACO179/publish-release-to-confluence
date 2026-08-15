# Publish to Confluence GitHub Action

- [Action Proposal](#action-proposal)
- [Setup](#setup)
- [Inputs](#inputs)
- [Usage](#usage)
- [To Do](#to-do)

## Action Proposal

This GitHub Action, named "Publish to Confluence", is designed to create a release page in Confluence with the release notes. The action fetches the release notes from a GitHub release and then posts them to a specified Confluence page.
To work properly, You need to have well structured release notes in your GitHub release. The release notes should be in markdown format.
![Release page example](media/Release_Confluence.png)

## Setup

1. Create API token in Confluence:
   - Go to your Atlassian account settings.
   - Click on "Security" in the left-hand menu.
   - Click on "Create and manage API tokens".
   - Click on "Create API token".
   - Enter a label for your token.
   - Click on "Create".
   - Copy the token and save it in a secure location.  
     Doc: <https://support.atlassian.com/atlassian-account/docs/manage-api-tokens-for-your-atlassian-account/>
     ![API Tokens](media/Atlassian_account_api_token.png)
2. Add the API token to your GitHub repository secrets:
   - Go to your GitHub repository.
   - Click on "Settings".
   - Click on "Secrets and variables" > "Actions".
   - Click on "New repository secret".
   - Enter "CONFLUENCE_API_TOKEN" in the "Name" field.
   - Paste the API token in the "Value" field.
   - Click on "Add secret".
   ![Action Secrets](media/Actions_secrets.png)
3. Add the email address associated with your Confluence account to your GitHub repository secrets:
    - Go to your GitHub repository.
    - Click on "Settings".
    - Click on "Secrets and variables" > "Actions".
    - Click on "Variables".
    - Click on "New repository variable".
    - Enter "CONFLUENCE_EMAIL" in the "Name" field.
    - Enter the email address associated with your Confluence account in the "Value" field.
    - Click on "Add variable".
4. Create a Confluence page to serve as the parent page for the release pages and note down the page ID (`parentId`).  
`https://<cloud-id>.atlassian.net/wiki/spaces/<space>/pages/<parentId>/<title>`  
5. Get the space ID of the Confluence space where the release page will be created:
   - Go to the Confluence space.
   - Click on "Space settings" > "Space details".
   - Note down the space `Key` from the URL:
   `https://<cloud-id>.atlassian.net/wiki/spaces/viewspacesummary.action?key=<space-key>`
   - Sent the space key to the following API to get the space ID:
   ```bash
    curl --request GET \
    --url 'https://<cloud-id>.atlassian.net/wiki/rest/api/space/<space-key>' \
    --user 'email@example.com:<api_token>' \
    --header 'Accept: application/json'
   ```
   200 Response
   ```json
    {
        "id": "123456789", << This is a spaceId
        "key": "<space-key>",
        "name": "Space Name",
        "type": "global",
        "status": "current",
        ...................
    }
   ```

## Inputs

The action requires the following inputs:

| Name                 | Description                                                                 | Required | Default                           |
|----------------------|-----------------------------------------------------------------------------|----------|-----------------------------------|
| `spaceId`            | The ID of the Confluence space where the release page will be created.      | True     | -                                 |
| `status`             | The status of the page to be created.                                       | True     | -                                 |
| `title`              | The title of the page to be created.                                        | True     | -                                 |
| `parentId`           | The ID of the parent page under which the new page will be created.         | True     | -                                 |
| `ConfluenceBaseUrl`  | The base URL of your Confluence instance.                                   | True     | -                                 |
| `tag`                | The tag of the GitHub release from which the release notes will be fetched. | True     | -                                 |
| `confluence_email`   | The email address associated with your Confluence account.                  | True     | -                                 |
| `confluence_api_token`| The API token for your Confluence account.                                  | True     | -                                 |
| `appName`            | The name of the application for which the release is being made.            | True     | -                                 |
| `ConfluenceSpaceKey` | The key of the Confluence space where the release page will be created.     | False     | -                                 |
| `repoOwner`          | The owner of the GitHub repository from which the release notes will be fetched. | True | -                                 |
| `repoName`           | The name of the GitHub repository from which the release notes will be fetched. | True | -                                 |
| `attachAssets`       | Attach the release's built artifacts to the page, not just its notes.       | False    | `false`                           |
| `assetPattern`       | Glob selecting which assets to attach (`gh release download --pattern`).    | False    | `*`                               |
| `githubToken`        | Token used to download assets and read the documents. Required when `attachAssets` is true or `docsParentId` is set. | False | - |
| `restrictUpdates`    | Restrict editing of created pages to the publishing account; reading unchanged. | False | `false`                       |
| `checksumFile`       | Checksum file whose contents are shown as text on the release page.         | False    | `SHA256SUMS`                      |
| `docsParentId`       | Parent page under which each matched Markdown file becomes a child page.    | False    | -                                 |
| `docsPaths`          | Comma-separated globs selecting which Markdown files to publish.            | False    | `README.md`                       |
| `docsTitlePrefix`    | Prefix added to every page title, to keep two repositories' pages apart.    | False    | -                                 |
| `docsTitleStrip`     | Prefix removed from a document's path before it becomes a page title.       | False    | -                                 |
| `publishReleaseNotes`| Create the release page. False publishes documentation only.                | False    | `true`                            |
| `ref`                | Git ref the documents are read from. Defaults to `tag`.                    | False    | -                                 |

### Attaching release assets

With `attachAssets: true` the action downloads the release's artifacts and adds
them to the page it just created. This is aimed at a **private** repository whose
users are not all on GitHub: colleagues open the Confluence page and download the
build without needing an account or repository access.

```yaml
    attachAssets: true
    assetPattern: "*.html"          # or "*" for everything
    githubToken: ${{ github.token }}
```

Notes:

- `githubToken` must be able to read the release. When the release lives in the
  same repository as the workflow, `${{ github.token }}` is enough.
- Attachment upload uses the **v1** API
  (`/wiki/rest/api/content/{id}/child/attachment`) even though page creation here
  uses v2, and it requires the `X-Atlassian-Token: no-check` header — without it
  Confluence rejects the POST as CSRF.
- Confluence enforces a maximum attachment size (100 MB on Cloud by default,
  though administrators often lower it). A rejected upload fails the step and
  prints the message Confluence returned.
- The action creates a **new page per release**, so attachments live on that
  release's page and the download link changes each time. If you need a link that
  does not move, point readers at the parent page.

### Mirroring the documentation

Set `docsParentId` and every Markdown file matched by `docsPaths` — as it stood
at the released tag, not on the default branch — is published as a child page
under it, titled after the file.

```yaml
    docsParentId: "123456789"
    docsPaths: "README.md,docs/*.md"
    githubToken: ${{ github.token }}
```

Page titles are the document's path without the `.md` extension, because a
repository of playbooks has a `README.md` in every directory and they cannot all
be one page. `docsTitleStrip` removes a leading prefix, so
`playbooks/deploy-certs/README.md` becomes `deploy-certs/README` rather than
carrying a directory nobody needs to read.

`docsTitlePrefix` does the opposite, and for the opposite reason: titles are
unique per Confluence space, so two repositories both publishing a `README`
collide. A prefix of `myapp/` gives `myapp/README` beside another project's.

### Publishing documentation without a release

A repository with no releases sets `publishReleaseNotes: false` and a `ref`:

```yaml
    publishReleaseNotes: "false"
    ref: main
    docsParentId: "123456789"
    docsPaths: "README.md,playbooks/*/README.md"
    docsTitleStrip: "playbooks/"
```

The release-notes and asset steps are skipped entirely, so no tag is needed.

> **Each child page's body is replaced on every release**, so those pages must
> exist only for this. The parent is never written to, so it is free to carry an
> index by hand.

A `*` does not cross a directory separator and patterns anchor at the repository
root, so `docs/*.md` takes `docs/frontend.md` and leaves `docs/agents/domain.md`
alone, and `README.md` cannot also match `docs/README.md`. Use `docs/**/*.md` to
reach into subdirectories.

Confluence page titles are unique per space, and names like `architecture` are
generic enough to already belong to someone else. A page with a matching title
found under a **different** parent stops the run rather than being overwritten.

The Markdown is rendered by GitHub's own `/markdown` endpoint, so tables, fenced
code and reference links come back correct, and the result is then reduced to the
XHTML subset Confluence's storage format accepts. Two consequences worth knowing:

- **Images are removed.** Status badges are typically served from endpoints that
  require repository access, so for readers who have none — the people this page
  exists for — they would render as broken images. Their surrounding links are
  dropped with them rather than left behind as empty anchors.
- **Heading self-links are unwrapped**, since the `#anchor` targets GitHub adds do
  not exist on a Confluence page.

The published page opens with a line stating which repository and tag it came
from, and that edits made in Confluence are overwritten by the next release.

**Mermaid diagrams are drawn, not quoted.** GitHub renders mermaid in the
browser rather than in its Markdown API, so a fence would otherwise arrive as a
code block and stay one — the source of a diagram instead of a diagram. Each
fence is lifted out before rendering, drawn with `@mermaid-js/mermaid-cli`,
attached to the page, and referenced with an `<ac:image>`. That needs the page
to exist first, so a document containing diagrams is published twice: once with
placeholders, then again with the images in place.

## Usage

To use this action in your GitHub workflow, add the following step:

```yaml
- name: Publish to Confluence
  uses: JACO179/publish-release-to-confluence@main
  with:
    spaceId: '<spaceId>'
    status: '<status>'
    title: '<title>'
    parentId: '<parentId>'
    ConfluenceBaseUrl: '<ConfluenceBaseUrl>'
    tag: '<tag>'
    confluence_email: '<confluence_email>'
    confluence_api_token: '<confluence_api_token>'
    appName: '<appName>'
    ConfluenceSpaceKey: '<ConfluenceSpaceKey>'
    repoOwner: '<repoOwner>'
    repoName: '<repoName>'
```

This fork cuts no releases of its own, so `@main` is the ref to track. Pin a
commit SHA instead if you need a target that does not move under you.

Example:

```yaml
  create-confluence-page:
    runs-on: ubuntu-latest
    needs: [ release ]
    steps:
      - name: Publish to Confluence Action
        uses: JACO179/publish-release-to-confluence@main
        with:
          confluence_email: ${{ vars.CONFLUENCE_EMAIL }}
          confluence_api_token: ${{ secrets.CONFLUENCE_API_TOKEN }}
          spaceId: '223903751'
          status: 'current'
          title: "Release: ${{ github.ref_name }}"
          parentId: '556597266'
          ConfluenceBaseUrl: 'https://smart-cooking.atlassian.net'
          repoOwner: ${{ github.repository_owner }}
          repoName: ${{ github.event.repository.name }}
          tag: ${{ github.ref_name }}
          appName: ${{ github.event.repository.name }}

```

    Replace the placeholders (<...>) with your actual values.

## To Do

1. Add support for customizing the release notes template.
2. Add support for customizing the page template.
3. Add support for page labels.
