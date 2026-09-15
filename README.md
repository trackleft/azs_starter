# azs_starter

A [Drupal recipe](https://www.drupal.org/docs/extending-drupal/drupal-recipes) that ships real Arizona Quickstart starter content — pages, Paragraphs (with their actual body content), media, and files — so a fresh Quickstart site doesn't start out empty.

Content was captured with Drupal core's `content:export` command and applied to this repo's `content/` directory, in the format core's own recipe importer understands natively (`_meta` + `default` per entity, keyed by UUID). Applying it needs nothing beyond Drupal core itself — no extra modules, no patches.

## Applying this recipe to a fresh local Quickstart install

These steps assume a local [Lando](https://lando.dev/) + [Arizona Quickstart](https://github.com/az-digital/az_quickstart) setup, e.g. via [az-quickstart-pantheon](https://github.com/az-digital/az-quickstart-pantheon). Any local Drupal + az_quickstart environment on **core 11.2 or later** works the same way.

1. **Get a fresh, installed Quickstart site.** For example, from an `az-quickstart-pantheon`-based project:

   ```bash
   lando composer install
   lando drush site-install az_quickstart -y
   ```

2. **Clone this repo somewhere Drupal can read it** — either directly into the site's own `recipes/` directory (which is gitignored by default in az-quickstart-pantheon projects, so it's safe to clone into), or anywhere else on disk:

   ```bash
   git clone https://github.com/trackleft/azs_starter.git recipes/azs_starter
   ```

3. **Apply the recipe.** Use an absolute, container-side path — `lando drush recipe` reliably fails to resolve a relative path (`recipes/azs_starter`) against a Pantheon-recipe Lando app, reporting it "is not a directory" even though it exists:

   ```bash
   lando drush recipe /app/recipes/azs_starter
   ```

4. **Rebuild cache and take a look:**

   ```bash
   lando drush cr
   ```

   You should now have the Quickstart demo pages (Home, News, Events, Directory, Wilbur/Wilma Wildcat profiles, etc.), each with their real Paragraph-based body content and images, not just empty shells.

**Recipes are a one-time, create-only operation.** They create new entities by UUID; they don't sync or update entities that already exist. So if you apply this recipe and it later gets new/changed content, re-running `drush recipe` on the *same* site won't pick up the changes — the importer will skip (or error on) anything whose UUID it already finds. To pick up an update on a site that already has this content applied, either delete the previously-imported entities (by UUID) first and re-apply, or apply the recipe to a fresh site instead.

## Updating this recipe (for maintainers)

If the source site's content has changed and you need to refresh what's shipped here:

1. The source site needs **Drupal core 11.3+** (for the `content:export` command) and a patched `drupal/entity_reference_revisions` so Paragraphs export as real, separate portable entities instead of dangling local IDs. Add this to the site's `composer.json`:

   ```bash
   composer config extra.patches.drupal/entity_reference_revisions --json '{"Support core content:export for Paragraphs (MR #69)": "https://git.drupalcode.org/project/entity_reference_revisions/-/merge_requests/69.diff"}'
   composer update drupal/entity_reference_revisions --with-all-dependencies
   ```

   This patch is only needed on the *exporting* site — sites that just apply the resulting recipe don't need it, since core's recipe importer already supports the UUID-based entity references the patch produces.

2. Export the content (adjust node IDs/bundles as needed):

   ```bash
   lando drush content:export node --with-dependencies --dir=/app/content-export
   ```

3. **Scrub real user data before committing anything.** `--with-dependencies` pulls in author accounts as `user` entities, which include real names, emails, and password hashes — none of that belongs in a public recipe. Rewrite every `uid`/`revision_uid` field from `entity: <uuid>` to `target_id: 1` (the conventional admin account, which exists on every Drupal site), strip the corresponding entries from each file's `_meta.depends`, and delete the exported `user/` directory entirely.

4. Replace this repo's `content/node/`, `content/paragraph/`, `content/media/`, and `content/file/` directories with the cleaned export.

5. Commit and push:

   ```bash
   git add recipe.yml content/
   git commit -m "Update starter content"
   git push origin main
   ```

   Since old and new UUIDs may not line up (a re-export can pick up different/renamed content), diff the changed files before committing rather than assuming a clean replace — and double check for real user data again, since it's easy to reintroduce by re-running the export.
