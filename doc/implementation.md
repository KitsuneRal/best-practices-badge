# Implementation

<!-- SPDX-License-Identifier: (MIT OR CC-BY-3.0+) -->

We have implemented a simple web application called "BadgeApp" that
quickly captures self-assertion data, evaluates criteria automatically
when it can, and provides badge information.
Our emphasis is on keeping the program relatively *simple*.

This file provides information on how it's implemented, in the hopes that
it will help people make improvements.
See [CONTRIBUTING.md](../CONTRIBUTING.md) for information on how to
contribute to this project, and [INSTALL.md](INSTALL.md) for information
on both how to install this software (e.g., for development) and a
"quick start" guide to getting something to happen.

In this document we'll use the term "open source software" (OSS),
and treat Free/Libre and Open Source Software (FLOSS) as a synonym.

## Requirements

The BadgeApp web application MUST:

1. Meet its own criteria.  This implies that it must be open source software
   (OSS).
2. Be capable of being developed and run using *only* OSS.
   It may *run* on proprietary software; portability improvements welcome.
   It's also fine if it can use proprietary services, as long as it can
   *run* without them.
3. Support users of modern widely used web browsers, including
   Chrome, Firefox, Safari, and Internet Explorer version 10 and up.
   We expect Internet Explorer pre-10 users will use a different browser.
4. NOT require JavaScript to be enabled on the web browser (some
   security-conscious people disable it) - instead, support graceful
   degradation (many features will work much better if JavaScript is enabled).
   Requiring CSS is fine.
5. Support users of various laptops/desktops
   (running Linux (Ubuntu, Debian, Fedora, Red Hat Enterprise Linus),
   Microsoft Windows, or Apple MacOS) and mobile devices
   (including at least Android and iOS).
6. NOT require OSS projects to use GitHub or git.
   We do use GitHub and git to *develop* BadgeApp (that's different).
7. Automatically fill in some criteria where it can, at least if a
   project is on GitHub.  Automating filling in data is a never-ending
   process of refinement. Thus, we intend to fill a few to start, and then
   add more automation over time.
8. Be secure.  See the separate
   [security](security.md) document for more about security, including
   its requirements.
9. Be accessible.
   We strive to comply with the
   <a href="https://www.w3.org/TR/WCAG20/">Web Content Accessibility
   Guidelines (WCAG 2.0)</a> (especially at the A and AA level).

There are many specific requirements; instead of a huge requirements document,
most specific requirements are proposed and processed via its
[GitHub issue tracker](https://github.com/coreinfrastructure/best-practices-badge/issues).
See [CONTRIBUTING](../CONTRIBUTING.md) for more.

## High-level architecture

The web application is itself OSS, and we intend for the
web application to meet its own criteria.
We have implemented it with Ruby on Rails; Rails is good for
simple web applications like this one.
We are currently using Rails version 4.
The production system stores the data in Postgres.

### High-level design figure

The following figure shows a high-level design of the implementation:

![Design](./design.png)

### Key components

Some other key components we use are:

- Bootstrap
- Jquery
- Jquery UI
- Imagesloaded <https://github.com/desandro/imagesloaded>
  (to ensure images are loaded before displaying them)
- Puma as the webserver - not webrick, because Puma can handle multiple
  processes and multiple threads.  See:
  <https://devcenter.heroku.com/articles/ruby-default-web-server>
- A number of supporting Ruby gems (see the Gemfile)

### Key classes

The software is designed as a traditional model/view/controller (MVC)
architecture.  As is standard for Rails, under directory "app"
(application) are directories for "models", "views", and "controllers".

Central classes include:

* "Project" (defined in file "app/models/project.rb")
  defines the model that captures data about a project.
* "User" (defined in file "app/models/user.rb")
  defines the model that captures data about a user.

## Deployment

We have three publicly accessible tiers:

* master - an instance of the master branch
* staging
* production

These are currently executed on Heroku.
If you have write authorization to the GitHub repository,
the commands "rake deploy_staging" and "rake deploy_production"
will update the staging and production branches (respectively).
Those updates will trigger tests by CircleCI (via webhooks).
If those tests pass, that updated branch is then deployed to
its respective tier.

Most administrative actions require logging into the relevant Heroku tier
using the "heroku" command (this requires authorization).
The one exception: the BadgeApp web application does support an 'admin'
role for logged in users; admin users
are allowed to edit and delete any project entry.

## Environment variables

The application is configured by various environment variables:

* PUBLIC_HOSTNAME (default 'localhost')
* BADGEAPP_MAX_REMINDERS (default 2): Number of email reminders to send
  to inactive projects when running "rake reminders".
  This rate limit is best set low to start,
  and relatively low afterwards, to limit impact if there's an error.
* LOST_PASSING_REMINDER (default 30): Minimum number of days since
  last lost a badge before sending reminder
* LAST_UPDATED_REMINDER (default 30): Minimum number of days
  since project last updated before sending reminder
* LAST_SENT_REMINDER (default 60): Minimum number of days since
  project was last sent a reminder
* RAILS_ENV (default 'development'): Rails environment, one of
  'test', 'development', 'fake_production', and 'production'.
  The master, staging, and production systems set this to 'production'.
  See the discussion below about fake_production.
* BADGEAPP_DAY_FOR_MONTHLY: Day of the month to monthly activities, e.g.,
  send out monthly reminders.  Default 5.  Set to 0 to disable monthly acts.
* FASTLY_CLIENT_IP_REQUIRED: If present, download the Fastly list of
  client IPs, and only let those IPs make requests.  Enabling this
  counters cloud piercing.  This isn't on by default, but the environment
  variables are set on our tiers.
* DB_POOL: Set the number of connections the app can hold to the database.
  This is important for performance on Heroku; see:
  <https://devcenter.heroku.com/articles/concurrency-and-database-connections>.
  If unset, defaults to RAILS_MAX_THREADS + 1 for this app,
  because in addition to every web thread we occasionally fire a task to
  process occasional requests (such as the daily task).
  If RAILS_MAX_THREADS is not set, presume it is 5.
* RAILS_LOG_LEVEL: Rails log level used when RAILS_ENV is either
  "production" or "fake_production". Plausible values are
  "debug", "info", and "warn". Default is "info". See:
  <http://guides.rubyonrails.org/debugging_rails_applications.html>

This can be set on Heroku.  For example, to change the maximum number
of email reminders to inactive projects on production-bestpractices:

~~~~
heroku config:set --app production-bestpractices BADGEAPP_MAX_REMINDERS=5
~~~~

On Heroku, using config:set to set a value will automatically restart the
application (causing it to take effect).

The TZ (timezone) environment variable is set to ":/usr/share/zoneinfo/UTC"
on all tiers.  We want all logging to be done in UTC (because then moving
the servers has no affect on logs).  Using leading-colon helps performance
on many systems, especially many Rails systems (because it skips
many system calls), and it's easy enough to do.  More information is at
[How setting the TZ environment variable avoids thousands of system calls](https://blog.packagecloud.io/eng/2017/02/21/set-environment-variable-save-thousands-of-system-calls/).
This was implemented with:

~~~~
heroku config:set --app production-bestpractices TZ=:/usr/share/zoneinfo/UTC
~~~~

## Terminology

This section describes key application-specific terminology.

The web application tracks data about many FLOSS *projects*,
as identified and entered by *users*.

We hope that projects will (eventually) *achieve* a *badge*.
A project must *satisfy* (or "pass") all *criteria*
(singular: criterion) *enough* to achieve a badge.

The *status* of each criterion, for a given project, can be one of:
'Met', 'Unmet', 'N/A' (not applicable, a status that only some
criteria can have), and '?' (unknown, the initial state of all
criteria for a project).
Every criterion can also have a *justification*.
For each project the system tracks the criteria status,
criteria justification, and a few other data fields such as
project name, project description, project home page URL, and
project repository (repo) URL.

Each criterion is in one of four *categories*:
'MUST', 'SHOULD', 'SUGGESTED', and 'FUTURE'.
In some cases, a criterion may require some justification
or a URL in the justification to be enough to satisfy the criterion for
a badge.  See the [criteria](./criteria.md) or
application form for the current exact rules.
A synonym for 'satifying' a criterion is 'passing' a criterion.

We have an 'autofill' system that fills in some data automatically.
In some cases the autofill data will *override* human-entered data
(this happens where we're either confident in the data, and/or
the data is not available using a common convention
that are enforcing for purposes of the badge).
The autofill system uses the metaphor of *Detectives* that need
some inputs, analyze them,
and produce outputs (including confidence levels).
Detectives are managed by a *Chief* of detectives.

See below for more detail.

## Running locally

Once your development environment is ready, you can run the application with:

~~~~
rails s
~~~~

This will automatically set up what it needs to, and then run the
web application.
You can press control-C at any time to stop it.

Then point your web browser at "localhost:3000".

## Security

See the separate
[security](security.md) document for more about security.

## Interface

This is a relatively simple web application, so its
external interface is simple too.

It has a few common use cases:

- Users who want to get a new badge.
  They will log in (possibly creating an account first
  if they don't use GitHub), select "add a project".
  They will see a long HTML form, which they can edit and submit.
  They can always go back, re-edit, and re-submit.
- Others who want to see the project data.  They can just go to
  the project page (they don't need to log in) to see the data.
- Those who want to see the badge for a given project
  (typically because this is transcluded).
  They would 'get' the /projects/:id/badge(.:format);
  by default, they would get an SVG file showing the status
  (i.e., 'passing' or 'failing').

Its interface supports the following interfaces, which is enough
to programmatically create a new user, login and logout, create project
data, edit it, and delete it (subject to the authorization rules).
In particular, viewing with a web browser (which by default emits 'GET')
a URL with the absolute path "/projects/:id" (where :id is an id number)
will retrieve HTML that shows the status for project number id.
A URL with absolute path "/projects/:id.json"
will retrieve just the status data in JSON format (useful for further
programmatic processing).

~~~~
Verb   URI Pattern                        Controller#Action
GET    /projects/:id(.:format)            projects#show # .json supported.
GET    /projects/:id/badge(.:format)      projects#badge {:format=>"svg"}

GET    /projects(.:format)                projects#index
POST   /projects(.:format)                projects#create
GET    /projects/new(.:format)            projects#new
GET    /projects/:id/edit(.:format)       projects#edit
PATCH  /projects/:id(.:format)            projects#update
PUT    /projects/:id(.:format)            projects#update
DELETE /projects/:id(.:format)            projects#destroy

GET    /users/new(.:format)               users#new
GET    /signup(.:format)                  users#new
GET    /users/:id/edit(.:format)          users#edit
GET    /users/:id(.:format)               users#show

GET    /sessions/new(.:format)            sessions#new
GET    /login(.:format)                   sessions#new
POST   /login(.:format)                   sessions#create
DELETE /logout(.:format)                  sessions#destroy
GET    /signout(.:format)                 sessions#destroy
~~~~

This uses Rails' convention where
a 'get' of /projects/:id/edit(.:format)' is considered an edit;
this would normally create a CSRF vulnerability, but Rails automatiacally
inserts and checks for a CSRF token, countering this potential vulnerability.

## Search

The "/projects" URL supports various searches.
We reserve the right to change the details, but we do try
to provide a reasonable interface.
The following search parameters are supported:

* status: "passing" or "in_progress"
* gteq: Integer, % greater than or equal
* lteq: Integer, % less than or equal.  Can be combined with gteq.
* pq: Text, "prefix query" - matches against *prefix* of URL or name
* q: Text, "normal query" - match against parsed name, description, URL.
  This is implemented by PostgreSQL, so you can use "&amp;" (and),
  "|" (or), and "'text...*'" (prefix).
  This parses URLs into parts; you can't search on a whole URL (use pq).
* page: Page to display

See app/controllers/project_controllers.rb for how these
are implemented.

## Downloading the database

We encourage analysis of OSS trends.
We do provide some search capabilities, but for most analysis
you will typically need to download the database.
We can't anticipate all possible uses, and we're trying to keep the
software relatively small & focused.

You can download the project data in JSON and CSV format using typical
Rails REST conventions.
Just add ".json" or ".csv" to the URL (or include an Accept statement,
like "Accept: application/json", in the HTTP header).
You can even do this on a search if we already support the search (e.g.,
by name).  Similarly, you can download user data in JSON format using
".json" at the end of the URL.

There is a current technical limitation in that you must
request project and user data page-by-page.
This isn't hard, just provide a page parameter (e.g., "page=2").
This is because Rails does not stream JSON or CSV data by default,
so if we allowed this the application would download
the entire database into memory to process it.
Rails applications *can* stream data (there are even web pages explaining
how to do it), but the call for it is rare and there are some
complications in its implementation, so we just haven't implemented it yet.

So you can download the projects by repeatedly requesting this
(changing "1" into 2, 3, etc. until all is loaded):

> https://bestpractices.coreinfrastructure.org/projects.json?page=1

You can similarly load the user data starting from here:

> https://bestpractices.coreinfrastructure.org/users.json?page=1

To directly download the user list you must be logged in, and we
intentionally restrict the information we share about users.
We only provide basic public information such as name, nickname, and such.
In particular, we only provide email addresses to
BadgeApp administrators, because we value the privacy of our users.

As we note about privacy and legal issues,
please see our <a href="https://www.linuxfoundation.org/privacy">privacy
policy</a> and <a href="https://www.linuxfoundation.org/terms">terms of
use</a>.
All publicly-available non-code content is released under at least the
<a href="https://creativecommons.org/licenses/by/3.0/">Creative Commons
Attribution License version 3.0 (CC-BY-3.0)</a>;
newer non-code content is released under
CC-BY version 3.0 or later (CC-BY-3.0+).
If referencing collectively or
not otherwise noted, please credit the
"CII Best Practices badge contributors" and note the license.
You should also note the website URL as the data source, which is
<https://bestpractices.coreinfrastructure.org>.

If you are doing research, we urge you do to it responsibly and reproducibly.
Please be sure to capture the date and time when you began and completed
capturing this dataset
(you need both, because the data could be changing
while you're downloading it).
Do what you can to ensure that your research can be
[replicated](https://en.wikipedia.org/wiki/Replication_crisis).
Consider the points in
[Good Enough Practices in Scientific Computing](https://arxiv.org/pdf/1609.00037v2.pdf), a "set of computing tools and techniques
that every researcher can and should adopt."
For example, "Where possible, save data as originally generated (i.e.
by an instrument or from a survey."
These aren't requirements for using this data, but they are well
worth considering.

## Changing criteria

To modify the text of the criteria, edit these files:

- doc/criteria.md - Document
- criteria/criteria.yml - YAML file used by BadgeApp for criteria information.

If you're adding/removing fields (including criteria), be sure to also edit
app/views/projects/\_form.html.erb
(to determine where to display it).
You may also want to edit the README.md file, which includes a summary
of the criteria.

When adding or removing fields, or when renaming
a criterion name, you may need to edit the test creator db/seeds.rb file,
and you will certainly need to create a database migration.
The "status" (met/unmet) is the criterion name + "\_status" stored as a string;
each criterion also has a name + "\_justification" stored as text.
So every add, remove, or rename of a criterion involves changing
*two* fields in the database schema.
Here are the commands, assuming your current directory is at the top level,
EDIT is the name of your favorite text editor, and MIGRATION_NAME is the
logical name you're giving to the migration (e.g., "add_discussion").
By convention, begin a migration name with 'add' to add a column and
'rename' to rename a column:

~~~~
  rails generate migration MIGRATION_NAME
  git add db/migrate/*MIGRATION_NAME.rb
  $EDITOR db/migrate/*MIGRATION_NAME.rb
~~~~

Your migration file should look something like this if it adds columns
(where add_column takes the name of the table, the name of the column,
the type of the column, and then various options):

~~~~
class MIGRATION_NAME < ActiveRecord::Migration
  def change
    add_column :projects, :crypto_alternatives_status, :string,
               null: false, default: '?'
    add_column :projects, :crypto_alternatives_justification, :text
  end
end
~~~~

Similarly, your migration file should look something like this
if it renames columns:

~~~~
class Rename < ActiveRecord::Migration
  def change
    rename_column :projects,
                  :description_sufficient_status,
                  :description_good_status
    rename_column :projects,
                  :description_sufficient_justification,
                  :description_good_justification
  end
end
~~~~

In some cases it may be useful to insert SQL commands or do
other special operations in a migration.
See the migrations in the db/migrate/ directory for examples.

Once you've created the migration file, check it first by running
"rake rubocop".  This will warn you of some potential issues, and
it's much better to fix them early.
(You can't run just "rake", because that invokes "rake test", and
the dynamic tests in "rake test" won't work until you execute
the migration).

You can migrate by running:

~~~~
  $ rake db:migrate
~~~~

If it fails, you *may* need to use "rake db:rollback" to roll it back.

You may also need to modify tests in the tests/ subdirectory, or
modify the autofill code in the app/lib/ directory.

Be sure to "git add" all new files, including any migration files,
and then use "git commit" and "git push".

## Internationalization (i18n) and localization (l10n)

This application is "internationalized", that is, it allows
users to select their locale, and then presents information
(such as human-readable text) in the selected locale.
If no locale is indicated, 'en' (English) is used.

To learn more about Rails and internationalization, please read the
[Rails Internationalization guide](http://guides.rubyonrails.org/i18n.html).

We can *always* use help in localizing (that is, in providing translations
of text to various locales) - please help!

### Requesting the locale at run-time

Users indicate the locale via the URL.
The recommended form is at the beginning of that path, e.g.,
<https://bestpractices.coreinfrastructure.org/fr/projects/>
selects the locale "fr" (French) when displaying "/projects".
This even works at the top page, e.g.,
<https://bestpractices.coreinfrastructure.org/fr/>.
It also supports the locale as a query parameter, e.g.,
<https://bestpractices.coreinfrastructure.org/projects?locale=fr>

### Fixing locale data

Almost all locale-specific data is stored in the "config/locales"
directory (one file for each locale, named LOCALE.yml). This data is
automatically loaded by Rails.  A few of the static files are served
directly, with a separate file for each locale;
see the "app/views/static_pages" directory.
If you need to fix a translation, that's where the data is.

### Adding a new locale

To add a new locale, modify the file "config/initializers/i18n.rb"
and edit the assignment of "I18n.available_locales" to add
the new locale.  The system will now permit users to request it.

Next, create a stub locale file in the "config/locales" directory
named LOCALE.yml.  A decent way to start is:

~~~~
cd config/locales
cp en.yml NEW_LOCALE.yml
~~~~

Edit the top of the file to change "en:" to your locale name.

At one time we suggested going to this page to get locale information
for Rails built-ins, and including that:
<https://github.com/svenfuchs/rails-i18n/blob/master/rails/locale/>
However, we now include the gem 'rails-i18n', and that provides
the same kind of functionality while being easier to maintain.

Now create the matching static pages in the
the "app/views/static_pages"  (cd "../..", then cd "app/views/static_pages",
then create the files using the new locale name).
Be sure all hypertext links to locations within the application
include the locale (e.g., use "/fr/projects" not "/projects" as the URL);
automatically-generated URLs include the locale, but constant strings do not.

Now the hard part: actually translating.
Edit the '.yml' (YAML) file to create the translations.
As always, you need to conform to YAML syntax.
For example,
strings that end in a colon (":") *must* be escaped (e.g., by
surrounding them with double-quotes).
Keys are *only* in lower case and never use dash (they use underscore).

Finally, you will need to update app/assets/javascripts/criteria.js.erb
to depend on the new locale's yml file, this allows the precompiler to be
to be notified if the contents of criteria.js should have changed.
Simply add the following line to the top of criteria.js.erb

~~~~
depend_on NEW_LOCALE.yml
~~~~

### Programmatically accessing a locale

To learn more about Rails and internationalization, please read the
[Rails Internationalization guide](http://guides.rubyonrails.org/i18n.html).

Inside views you can use the 't' helper, e.g.,

~~~~
    <%= t('hello') %>
    <%= t('.current_scope') %>
~~~~

Inside other code (e.g., in a flash message), use `I18n.t`:

~~~~
    I18n.t 'hello'
~~~~

You can access 'I18n.locale' to see the current locale's value
(this is a thread-local query, so this works fine when multiple
threads are active).

## App authentication via GitHub

The BadgeApp needs to authenticate itself through OAuth2 on
GitHub if users are logging in with their GitHub accounts.
It also needs to authenticate itself to get repo details from GitHub if a
project is being hosted there.
The app needs to be registered with GitHub[1] and its OAuth2 credentials
stored as environment variables.
The variable names of Oauth2 credentials are "GITHUB_KEY" and "GITHUB_SECRET".
If running on heroku, set config variables by following instructions on [2].

If running locally, these variables need to be set up.
We have set up a file '.env' at the top level which has stub values,
formatted like this, so that it automatically starts up
(note that these keys are *not* what we used for the deployed systems,
for obvious reasons):

~~~~sh
export GITHUB_KEY = '..VALUE..'
export GITHUB_SECRET = '..VALUE..'
~~~~

You can instead provide the information this way if you want to
temporarily override these:

~~~~sh
GITHUB_KEY='client id' GITHUB_SECRET='client secret' rails s
~~~~

where *client id* and *client secret* are registered OAuth2 credentials
of the app.

The authorization callback URL in GitHub is:
<http://localhost:3000/auth/github>

[1] <https://github.com/settings/applications/new>
[2] <https://devcenter.heroku.com/articles/config-vars>

## Database content manipulation

In some cases you may need to view or edit the database contents directly.
For example, we don't currently have code to set a user to have the
'admin' role, or to change the ownership of a project,
to backup the database, or restore the database.
Instead, we simply interact with the database software, which
already has the functions to do this.

In development mode, simply view 'localhost:3000/rails/db'
in your web browser.
You can select any table (on the left-hand side) so you can view
or edit the database contents with a UI.

You can directly connect to the database engine and run commands.
On the local development system, run "rails db" as always.
To change the database contents of a production system,
log into that system and use the SQL language to make changes.
E.G., on Heroku, presuming that you have installed the heroku command,
and configured it for the system you are controlling
(including the necessary keys),
you can pipe SQL commands to 'heroku pg:psql'.
This only works if you've been given keys to control this.
On Heroku we use PostgreSQL.
Here are a few examples (replace the "heroku pg:psql..." with "rails db"
to do it locally):

~~~~sh
echo "SELECT * FROM users WHERE users.id = 1" | \
  heroku pg:psql --app master-bestpractices
echo "SELECT * FROM users WHERE name = 'David A. Wheeler'" | \
  heroku pg:psql --app master-bestpractices
echo "UPDATE users SET role = 'admin' where id = 25" | \
  heroku pg:psql --app master-bestpractices
echo "UPDATE projects SET user_id = 25 WHERE id = 1" | \
  heroku pg:psql --app master-bestpractices
~~~~

You can force-create new users and make them admins
(again, if you have the rights to do so).
To create new github user, first get their github uid from their
github username (nickname) by looking at
<https://api.github.com/users/USERNAME>
and getting the "id" value.
Then run this, replacing all-caps stubs with the values in single quotes
(this will create a local id automatically):

~~~~sh
echo "INSERT INTO users (provider,uid,name,nickname,email,role,activated,
  created_at,updated_at)
  VALUES ('github',GITHUB_UID,FULL_USER_NAME,
  GITHUB_USERNAME,EMAIL,'admin',t,now(),now());" | \
  heroku pg:psql --app master-bestpractices
~~~~

You can
[import or export databases on Heroku](https://devcenter.heroku.com/articles/heroku-postgres-import-export)
For example, here's how to quickly back up the database
(presuming that it's set up for the Heroku site and that you have
the authorization keys to do this):

~~~~sh
heroku pg:backups capture
curl -o latest.dump $(heroku pg:backups public-url)
~~~~

## Recovering a deleted or mangled project entry

If you want to restore a deleted project, or reset its values,
we have some tools to help.

Put the project data in JSON form in the file "project.json"
(at the top of the tree, typically in "~/cii-best-practices-badge").
If this was a recent deletion, then you can simply copy the JSON-formatted
data from the email documenting the deletion.

Then run:

~~~~
    rake create_project_insertion_command
~~~~

This will create a file "project.sql" that has SQL insertion command.

You'll next need to delete the project if it already exists, because
it's an insertion command.

Now you need to execute the SQL command on the correct database.
Locally you can do this (you may want to set RAILS_ENV to
"production"):

~~~~
    rails db < project.sql
~~~~

If you want the data to be on the true production site, you'll need
privileges to execute database commands, then run this:

~~~~
    heroku pg:psql --app production-bestpractices < project.sql
~~~~

## Purging Fastly CDN cache

If a change in the application causes any badge level(s) to change or
changes the output of a projects json file,
you need to purge the Fastly CDN cache after pushing.
Otherwise, the Fastly CDN cache will continue to serve the old badge
images as well as project json files (until they time out).

You can purge the Fastly CDN cache this way (assuming you're
allowed to log in to the relevant Heroku app):

~~~~sh
heroku run --app HEROKU_APP_HERE rake fastly:purge
~~~~

This command will use the value of the FASTLY_API_KEY
configured for that Heroku application (Fastly requires authorization
for purging the entire cache)... so you don't have to provide it yourself.

It's safe to purge the cache if you're not sure if you need to do it.
After a cache purge, the next request for each badge will go
to the website, so for a brief time the site will
be busy serving badge files.

## Resetting Heroku plug-ins

Here's how to reset the heroku-local plugin:

~~~~sh
heroku plugins:uninstall heroku-local --app master-bestpractices
heroku plugins --app master-bestpractices
~~~~

The latter automatically reinstalls heroku-local.
This information is from: <https://github.com/heroku/heroku/issues/1690>.

Normally you should just push changes to "master" first, so that
CircleCI will test it.  If you want to push directly to Heroku
(and have the necessary rights):

~~~~
git remote add heroku https://git.heroku.com/master-bestpractices.git
~~~~

Now you can directly deploy to Heroku:

~~~~
git checkout master
git push heroku master
~~~~

## Auditing

The intent is to eventually have an "audit" function that
runs auto-fill without actually editing the results, and then
show the differences between the automatic results and the form values.
This will let external users compare things.

## Autofill

The process of automatically filling in the form is called
"autofill".

Earlier discussions presumed that the human would always be right, and
that the automation would only fill in unknowns ("?").
However, we've since abandoned this; instead, in some cases we want
to override (either because we're confident or because we want to require
projects to provide data in a way that we can be confident in it).

Autofill must use some sort of pluggable interface, so that people
can add them.  We will focus on getting data from GitHub, e.g.,
api.gihub.com/repos has a lot of information.
The pluggable interface could be implemented using Service Objects;
it's not clear that's the best way.
We do want to create directories where people can just add new files to
add new plug-ins.

We name each separate module that detects something a "Detective".
A Detective needs to be called, be able to get data, and eventually
return a set of findings.
The findings are a hash with
attributes and findings about them:
(proposed new) value, confidence, and justification (string).

The "Chief" module calls the Detectives in the right order and
merges the results.
Confidence values range from 0..5; confidence values of 4 or higher
override the user input.

## Authentication

Currently we allow people to log in using their GitHub account
or a local account (so people who don't want to use GitHub don't need to).
We trust GitHub's answers about whether or not a user is who they say they
are, and about which GitHub projects they can edit.

We currently can't be sure if a local user is actually allowed to
edit a given project, but admins can override any claims if necessary.
If this becomes a problem, we could make it possible for a
a project URL page to include the
token (typically in an HTML comment) to prove that a given user is
allowed to represent that particular project.
That would enable projects to identify users who can represent them
without requiring a GitHub account.

Future versions might support sites other than GitHub; the design should
make it easy to add other sites in the future.

We make public the *username* of who last
entered data for each project (generally that would be the GitHub username),
along with the edit time.

## Plans: Who can edit project P?

(This is a summary of the previous section.)

A user can edit project P if one of the following is true:

1. If the user is an "admin" then the user can edit the
  badge information about any project.
  This will let the Linux Foundation fix problems.
2. If project P is on GitHub AND the user is authorized via GitHub
  to edit project P, then that user can edit the badge information about
  project P.  In the future we might add repos other than GitHub, with
  the same kind of rule.
3. If the user created this badge entry, the user can edit it.

## GitHub-related badges

Pages related to GitHub-related badges include:

* <http://shields.io/> - serves files that display a badge
  (as good-looking scalable SVG files)
* <https://github.com/badges/shields> -  Shields badge specification,
  website and default API server (connected to shields.io)
* <http://nicbell.net/blog/github-flair> - a blog post that identifies
  and discusses popular GitHub flair (badges)

We want GitHub users to think of this
as &#8220;just another badge to get.&#8221;

We intend to sign up for a few badges so we can
evalute their onboarding process,
e.g., Travis (CI automation), Code Climate (code quality checker including
BrakeMan), Coveralls (code coverage), Hound (code style),
Gymnasium (checks dependencies), HCI (looks at your documentation).
For example, they provide the markdown necessary to embed the badge.
See ActiveAdmin for an example, take a few screenshots.
Many of these badges try to represent real-time status.
We might not include these badges in our system, but they
provide useful examples.

## Other badging systems

Mozilla's Open Badges project at <http://openbadges.org/>
is interesting, however, it is focused on giving badges to
individuals not projects.

## CircleCI

The CircleCI build execution is configured to use Ubuntu 14.04 (Trusty);
it was Ubuntu 12.04 (Precise).

## License detection

Some information on how to detect licenses in projects
(so we can perhaps autofill them) can be found in
[&#8220;Open Source Licensing by the Numbers&#8221; by Ben Balter](https://speakerdeck.com/benbalter/open-source-licensing-by-the-numbers).

For the moment, we just use GitHub's mechanism.
It's easy to invoke and resolves it in a number of cases.

## Implementation of Detectives.

The detective classes are located in the directory often located in the directory ./workspace/cii-best-practices-badge/app/lib.  This directory contains all of the detectives and has a very specific naming convention.  All new detectives must be named name1_detective.rb.  This name is important as it will be called by the primary code chief.rb which calls and collects the results of all of the detective classes.

To integrate a new class chief.rb must be edited in the following line.

ALL_DETECTIVES =
  [
    NameFromUrlDetective, ProjectSitesHttpsDetective,
    GithubBasicDetective, HowAccessRepoFilesDetective,
    RepoFilesExamineDetective, FlossLicenseDetective,
    HardenedSitesDetective (Name1Detective)
  ].freeze

  where Name1Detective corrosponds to the new class created in name1_detective.  Without following the naming convention chief will not run the new detective.

  A template detective called blank_detective.rb is supplied with the project with internal documentation as to how to use it.

  Remember, in addition to the detective you must right a test in order for it
  to be accepted into the repository.  The tests are located at ./test/unit/lib/
  with an example test of blank_detective included.

## Analysis

We use the OWASP ZAP web application scanner to find potential
vulnerabilities.
This lets us fulfill the "dynamic analysis" criterion.

## Setup for deployment

If you want to deploy this yourself, you need to set some things up.
Here we'll presume Heroku.

You need to have email set up.
See the Action mailer basics guide at
<http://guides.rubyonrails.org/action_mailer_basics.html>
and Hartl's Rails tutorial, e.g.:
<https://www.railstutorial.org/book/account_activation_password_reset#sec-email_in_production>

To install sendgrid on Heroku to make this work, use:

~~~~sh
heroku addons:create sendgrid:starter
~~~~

If you plan to handle a lot of queries, you probably want to use a CDN.
It's currently set up for Fastly.

## Badge SVG

The SVG files for badges are:

- <https://img.shields.io/badge/cii_best_practices-passing-green.svg>
- <https://img.shields.io/badge/cii_best_practices-in_progress-yellow.svg>
- <https://img.shields.io/badge/cii_best_practices-failing-red.svg>

## Licenses of the software used by BadgeApp

See CONTRIBUTING.md for the license rules;
fundamentally we require software to be released as OSS
before we can depend on it.

The following components don't declare a license in their Gemfile,
and were researched separately:

* gitlab: URL <https://github.com/NARKOZ/gitlab/blob/master/LICENSE.txt> reveals this to be license BSD-2-Clause.
* colored: URL <https://github.com/defunkt/colored/blob/master/LICENSE> reveals this to be license MIT.

For more on license decisions see doc/dependency_decisions.yml.
You can also run 'rake' and see the generated report
license_finder_report.html.

## HTML link checking

GitHub has relatively recently changed its robots.txt file so
that only certain agents are allowed to retrieve files.
This means that typical link-checking services don't work, since common
services like the W3C's link checker are rejected.

This can be worked around by downloading the W3C link checker,
disabling robots.txt, and running it directly.  You need to be very
careful when doing this.  We'll install the "Linkchecker" package from CPAN
(command name is 'checklink') to do this.  Here's how.

~~~~
cpan /W3C-LinkChecker-4.81/
cpan LWP::Protocol::https # Needed for HTTPS
su
cd /usr/local/bin
cp checklink checklink-norobots

patch -p0 <<END
--- checklink   2016-02-24 10:37:05.000000000 -0500
+++ checklink-norobots  2016-02-24 10:48:24.856983414 -0500
@@ -48,7 +48,7 @@
 use Net::HTTP::Methods 5.833 qw();    # >= 5.833 for 4kB cookies (#6678)

 # if 0, ignore robots exclusion (useful for testing)
-use constant USE_ROBOT_UA => 1;
+use constant USE_ROBOT_UA => 0;

 if (USE_ROBOT_UA) {
     @W3C::UserAgent::ISA = qw(LWP::RobotUA);
END
~~~~

You can then run, e.g.:

~~~~
checklink-norobots -b -e \
  https://github.com/coreinfrastructure/best-practices-badge | tee results
~~~~

## PostgreSQL Dependencies

As a policy, we minimize the number of dependencies on any particular
database implementation where we can.  Where possible, please
prefer portable constructs (such as ActiveRecord).

However, our current implementation requires PostgreSQL.  Our internal
project search engine uses PostgreSQL specific commands.  Additionally,
we are using the PostgreSQL specific citext character string type to
store email addresses.  This allows us, within PostgreSQL, to store
case sensitive emails but have a case insensitive index on them.

We do this as we can foresee a case where a user's email requires case
sensitivity to be received (Microsoft Exchange allows this).  We do not,
however, want to allow for emails that are not case insensitive unique
since this could possibly allow for a number of duplicate users to be
created and the possibility of two users from the same domain having
emails which differ only in case is exceedingly rare.

Using these PostgreSQL-specific capabilities makes the software much
smaller.  Limiting these dependencies makes it easier to port to a
different RDBMS if necessary.
Since PostgreSQL is itself OSS, this isn't as dangerous as becoming
dependent on a single supplier whose product cannot be forked.

## Accessing our analysis tools

We have various analyzers.  Here are some hints of how to access them.

If you don’t install the software, then you can use our REST interface
– create a project & then query what we’ve learned.  That only
provides *some* functionality.

If you *install* software, then there are many more options.  The software
was designed to be used as a website, so you can still use it that way,
but then you can also directly invoke the functionality you want via
Ruby and Rails.  It’s easy to integrate as a CLI that way.

We’re big on testing, for example, we have 100% statement coverage.
A side-effect of this is that a lot of functionality can be called
separately (so it can be tested).  You can also look at our tests to see
how to invoke something internally.  In particular, we have a number of
tools that try to gather data about a project – each one is called a
“Detective”, and they are managed by a “Chief” of Detectives.
Here’s a quick example that may help:

~~~~ruby
# Start up
rails console
p = Project.new
# Set values for project to evaluate.  We'll examine our own project.
p[:repo_url] = 'https://github.com/coreinfrastructure/best-practices-badge'
p[:homepage_url] = 'https://github.com/coreinfrastructure/best-practices-badge'
# Setup chief to analyze things:
new_chief = Chief.new(p, proc { Octokit::Client.new })
# Ask chief to find probable values:
results = new_chief.autofill

# Now "results" shows the fields found.
# For each field it has a value, confidence, and explanation.
# In addition, "p" is changed where we have high confidence.
results.keys
# => [:name, :sites_https_status, :repo_public_status, :repo_track_status,
# :repo_distributed_status, :contribution_status, :discussion_status,
# :license, :repo_files, :license_location_status, :release_notes_status,
# :floss_license_osi_status, :floss_license_status, :hardened_site_status,
# :build_status, :build_common_tools_status, :documentation_basics_status]
results[:name]
# => {:value=>"Core Infrastructure Initiative Best Practices Badge",
# :confidence=>3, :explanation=>"GitHub name"}
results[:license]
# => {:value=>"MIT", :confidence=>3,
# :explanation=>"GitHub API license analysis"}
p[:name]
# => "Core Infrastructure Initiative Best Practices Badge"
~~~~

## Forbidden Passwords

[NIST has proposed draft password rules in 2016](https://nakedsecurity.sophos.com/2016/08/18/nists-new-password-rules-what-you-need-to-know/).
They recommend having a minimum of 8 characters in passwords and
checking against a list of bad passwords.
Here we'll call them forbidden passwords - they are forbidden because
they're too easy to guess.

Here's how to recreate the bad-passwords list.
It's derived from the skyzyx "bad-passwords" list, which is dedicated
to the public domain via the CC0 license.

We create a modified version of the original source material.
We don't need to store anything less than 8 characters
(they will be forbidden anyway), and we only store lowercase versions
(we check downcased versions).
We compress it into a .gz file; it doesn't take long to read, and that greatly
reduces the space we use when storing and and transmitting the program.
Using the bad-passwords version dated "May 27 11:03:00 2016 -0700",
starting with the "mutated" list, we end up with 106,251 forbidden passwords.

~~~
(cd .. && git clone https://github.com/skyzyx/bad-passwords )
cat ../bad-passwords/raw-mutated.txt | grep -E '^.{8}' | tr A-Z a-z | \
  sort -u > raw-bad-passwords-lowercase.txt
rm -f raw-bad-passwords-lowercase.txt.gz
gzip --best raw-bad-passwords-lowercase.txt
~~~~

## Project stats omission on 2017-02-28

The production site maintains a number of daily statistics and can
[display the statistics graphically](https://bestpractices.coreinfrastructure.org/project_stats), but it is
missing a report for 2017-02-28.
This was due to a multi-hour downtime in
Amazon’s S3 web-based storage service, part of
Amazon Web Services (AWS), which took a large number of sites
(not just ours).
For more information you can see the story in
[USA Today](https://www.usatoday.com/story/tech/news/2017/02/28/amazons-cloud-service-goes-down-sites-scramble/98530914/),
[Zero Hedge](http://www.zerohedge.com/news/2017-02-28/amazon-cloud-reporting-increased-error-rates-secgov-possibly-impacted),
and
[Tech Crunch](https://techcrunch.com/2017/02/28/amazon-aws-s3-outage-is-breaking-things-for-a-lot-of-websites-and-apps/).

## fake_production

If you want to debug a problem that only appears in a production-like
envionment, try the 'fake_production' environment.
Here is how to enable it:

~~~~
RAILS_ENV=fake_production rails s
~~~~

This environment is almost exactly like production, with the
following differences:

* does not force HTTPS (TLS), so you can interact with it locally
* enables byebug so that you can insert breakpoints
* disables timeouts, so that you aren't rushed trying
  to track down a problem before the timeout ends.

Other environment variables might be usefully set in the command prefix,
such as "DATABASE_URL=development".

## See also

See the separate "[background](./background.md)" and
"[criteria](./criteria.md)" pages for more information.
