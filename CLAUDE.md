# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Lokka is a WordPress-inspired CMS written in Ruby (Sinatra 4 + ActiveRecord 8.1). Designed for cloud environments (Heroku, etc.) and local deployment. Ruby >= 3.2 required.

## Commands

```bash
# Install
bundle install

# Dev server (http://localhost:9292/)
bundle exec rackup

# Database
bundle exec rake db:setup      # migrate + seed
bundle exec rake db:migrate    # run migrations only
bundle exec rake db:seed       # seed only
bundle exec rake db:reset      # delete all + seed
bundle exec rake db:delete     # drop all tables + re-migrate

# Tests (Minitest + Rack::Test)
bundle exec rake               # default: db:delete db:setup test
bundle exec rake test           # run all tests
ruby -Ilib:test test/integration/admin/posts_test.rb              # single file
ruby -Ilib:test test/integration/admin/posts_test.rb -n test_name # single test

# DataMapper migration tools (one-time use)
bundle exec rake db:dump_dm SOURCE=path  # export DM data to db/dm_dump.json
bundle exec rake db:import_dm            # import into ActiveRecord

# Admin JS
bundle exec rake admin:install_deps   # npm install in public/admin
bundle exec rake admin:build_js       # build admin JavaScript
```

The default `rake` task **deletes the test database** before running. `RACK_ENV` and `LOKKA_ENV` are both set to `test`.

## Architecture

Entry point: `config.ru` -> `init.rb` -> `lib/lokka.rb` -> `Lokka::App` (Sinatra::Base subclass).

### Core (`lib/lokka/`)

- `app.rb` — Main Sinatra app. Configures sessions, CSRF, registers Padrino helpers, Sinatra::Flash, Lokka::Api, loads plugins, connects DB. Loads route files below.
- `app/entries.rb` — Public routes: index, Atom feed, search, category/tag/date archives, single entry, comments, sitemap.xml.
- `app/admin.rb` — `/admin/*` routes: CRUD for posts, pages, comments, categories, tags, users, snippets, field_names, themes, site settings, file uploads, preview.
- `app/api.rb` — REST API v1 (`/api/v1/*`). Bearer token auth. Full CRUD for posts, pages, categories, tags, comments, snippets, users (read-only), site settings. `POST /api/v1/token` for token generation.
- `before.rb` — `before` filter: locale detection, theme resolution, loads `@site` and common collections for all templates. Enforces login for `/admin/*` (except login page).
- `database.rb` — `Lokka::Database`: ActiveRecord connection, `migrate`/`migrate!`/`seed`. Reads `database.yml` or `database.default.yml` (ERB-evaluated).
- `helpers/helpers.rb` — Auth helpers (`login_required`, `current_user`, `logged_in?`), template type predicates (`index?`, `post?`, etc.), `bread_crumb`, `category_tree`, `comment_form`, admin entry CRUD helpers, file upload/preview.
- `helpers/render_helper.rb` — `render_any`/`render_detect`/`partial`: tries each supported template engine (ERB, HAML) to find a matching view.
- `importer.rb` — WordPress WXR importer.

### Models (`lib/lokka/models/`)

All ActiveRecord. Key patterns:
- **Entry** (STI) — `Post` and `Page` subclasses. Custom fields via `method_missing` delegating to `Field` records. Tags via `Tagging` join table.
- **Site** — Singleton. Dynamic option access via `method_missing` reading from `Option` table.
- **Option** — String primary key (`name`). Key-value config store with dynamic `method_missing` access.
- **Category** — Self-referential hierarchy (`parent_id`). `has_many :children`.
- **User** — SHA1(password+salt) auth. `api_token` for REST API. `admin?` role check.
- **Comment** — Statuses: MODERATED(0), APPROVED(1), SPAM(2).
- **Tag/Tagging** — Polymorphic tagging.

### Migrations (`db/migrate/`)

- `001_create_tables.rb` — Full schema (all tables).
- `002_add_api_token_to_users.rb` — API token column.

### Themes and Plugins

- `public/theme/<name>/` — Themes need at least `entries.erb` and `entry.erb` (ERB/HAML supported). Active theme set via `Site.theme`, resolved per-request in `before.rb`.
- `public/plugin/lokka-<name>/` — Sinatra extensions. Define `module Lokka::Foo; def self.registered(app); ...; end; end` at `public/plugin/lokka-foo/lib/lokka/foo.rb`. Plugin i18n files auto-loaded from `i18n/` subdir.
- `public/admin/` — Admin panel templates (ERB/HAML).

### Custom Permalinks

`not_found` in `app.rb` delegates to `custom_permalink?`/`custom_permalink_fix`/`custom_permalink_entry` helpers before rendering 404. Permalink settings managed in admin.

## Testing

- **Framework**: Minitest 5 + Rack::Test + FactoryBot + DatabaseCleaner (truncation)
- **Factories**: `test/factories.rb`
- **Test base class**: `LokkaTestCase` (includes Rack::Test::Methods, FactoryBot, Lokka::Helpers)
- **Support**: `test/support/` — `InSiteContext` (creates default Site), `AdminLoginContext` (creates user + logs in)
- **Integration tests**: `test/integration/` (admin and public routes)

## Database Config

`database.default.yml` (copy to `database.yml` to customize):
- Development/Test: SQLite3 (`db/development.sqlite3`, `db/test.sqlite3`)
- Production: `ENV['DATABASE_URL']` (PostgreSQL)
