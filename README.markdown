# John's Island Real Estate (JIRE2)

This is the documentation for [the John's Island Real Estate project](http://www.johnsislandrealestate.com). 

The repo name is based on this being the second CMS for the site.


## Developers

### Setup

1. Make sure you're a team memember on the Github repo and have access to the Heroku app. Email cody@gunnertech.com if you don't
2. Pull down the repo from your rails directory

  ```
  git clone git@github.com:gunnertech/jire2.git
  ```

3. Change into the jire directory

  ```
  cd jire2/
  ```

4. Install the needed ruby

  ```
  rvm install ruby-2.2.1
  ```
    
5. Generate the gemset

  ```
  cd .. && cd jire2
  ```

6. Install bundler

  ```
  gem install bundler --no-ri --no-rdoc
  ```

7. Install the gems

  ```
  bundle install
  ```
    
8. Setup the database using a database.yml file for postgres

  ```
  cp config/database.yml.example config/database.yml
  # change username to your postgres username
  ```
    
9. Connect the app to heroku

  ```
  heroku git:remote -a jire -r production
  heroku git:remote -a jire-staging -r staging
  ```
    
10. Setup your environment variables

  ```
  heroku config -r production -s  >> .env
  # open .env and change "production" to "development" (TODO: script this)
  ```
    
11. Pull down heroku's database to local

  ```
  foreman run bundle exec rake db:sync
  ```
     
12. Start the app

  ```
  foreman start
  ```  
     
### Deploying

#### Production
```
git add .; git commit -am "Fixes/closes/whatever ticket #<ticket-number>"; git push origin master; git push production master;
```

#### Staging
```
git add .; git commit -am "Fixes/closes/whatever ticket #<ticket-number>"; git push origin master; git push staging master;
```

Note that in both cases ```master``` was used as the working branch. If you're deploying another branch (usually to staging) it will look like this.

```
git add .; git commit -am "Fixes/closes/whatever ticket #<ticket-number>"; git push origin master; git push staging master:<branch>;
```


### Troubleshooting

The most common issues reside around two features:

#### Syncing with Property Base

Whenever a listing is saved in Refinery, the app (via lifecycle hooks after_save, after_update, after_create, etc) sends the new data to Property Base via Saleforce's REST API (using the Restforce gem).

It does so via Delayed Job in the background.

On Property Base, both a Listing Object and a Property Object get updated with the Sync. (See Integrations for more).

It's kind of a fire and forget, so the app does not provide feedback on whether the sync was successful.

You can manually check it by going to Property Base and searching for the listing and property object to make sure they got updated.

Additionally, you can trace some issues by opening a console ```heroku run rails console -r production``` and running something like ```listing.sync_listing_with_property_base_without_delay``` this will run the listing sync immediately and allow you the see the output if you're tailing the logs ```heroku logs --tail -r production```

#### Email notifications

For much of the same reasons that troubleshooting PB sync is difficult, email notification debugging is also difficult - mainly the use of Delayed Job.

In both these situations, we sometimes have issues where either the users are not notified when they should be or are notified multiple times.

This hasn't come up in a while, but the only way to try and glean why is to use papertrail (https://github.com/airblade/paper_trail) on the listing in question and decipher what happend.

There are two ways the app sends out email notifcations:

##### Favorite Listings

Favorite listings is straight forward.

A user sees a listing and wants to know if and when anything about it gets updated, so they "Favorite" that listing, which is akin to watching it.

Using lifecycle hooks, if the listing is updated via Refinery, we notify any users who have favorited that listing via email.

##### Saved Searches

Saved searches is akin to creating buckets.

So, for example, a user can create a bucket for all listings that are greater than $1 million.

If any listing is added to that bucket (meaning a new listing is created that tops $1 mil or an existing listing is updated with a price that exceeds that mark), all users who created that bucket are notified.

### Environments

#### Development

Each developer has their own development environment, which is used to build new features and fix bugs.

Unless you edit the .env file, you will access the the development environment in the browser at http://localhost:5000/.

To start the server, just make sure you're in the project directory and run:

```
foreman start
```

Likewise, you can fire up console with:

```
foreman run bundle exec rails console
```

The development environment does use Delayed Job.

#### Staging

The staging environment is publicly accessible and is used for demonstrating new features to the client prior to production release.

The general protocol is to use separate topic branches for each feature and push those branches to the staging server where the client can approve the work.

Once approved, the topic branch is merged into the master branch and deployed to production.

The staging environment is hosted on Heroku with the app name "jire-staging."

You can access it the browser here: https://jire-staging.herokuapp.com/

The staging environment only has one web instance and no worker instances, therefore it does not use Delayed Job.

#### Production

Like the staging environment, the production environment is hosted on Heroku, but with much better specs.

The production environment is accessed in the browser here www.johnsislandrealestate.com

Although this is subject to change, the production environment operates off of two web dynos and a single worker dyno (which powers Delayed Job for background tasks).

Aside from these, the production environment also uses the following resources:

##### Heroku Postgres

The Postgres Database. If you need to know what the DB is, stop reading this and go put your head down.

##### Logentries

This addon is used for log management, specifically, we use it to monitor the logs for dynos that have gone off line and restart them if they have. See ```restart_app_job.rb``` and ```alerts_controller.rb``` for more info on how these work. Logentries makes an HTTP request to alerts_controller when matching conditions occur.

##### Heroku Scheduler

This is basically our cron tab and it runs one job every night ```run_nightly_tasks```

This job has two methods:

1. ```update_statuses``` which handles changing listing statuses automatically if set to do so via Refinery
2. ```upload_to_luxury_realestate``` which FTPs our listings to Luxury Real Estate

##### MemCachier

Used for Rails cache to increase page load times

##### New Relic APM

Used for performance and error monitoring, although we don't use it nearly as much as we should.

### Integrations

Below is a list of integrations. 

See the Credentials section for access information to each of these.

#### Refinery

Refinery is the CMS that JIRE Staff (mainly Robyn) uses to update the site.

Updates include, but aren't limited to:

1. Adding new listings
2. Adding/removing photos to existing listings
3. Editing listings such as price and status
4. Editing pages, most importantly the home page

You access refinery via /refinery (so production would be http://www.johnsislandrealestate.com/refinery, etc).

Generally, all management is done under the "admin" username.

#### Amazon Web Services

The app uses AWS's S3 and Cloudfront services for two purposes:

1. Storing and hosting uploaded/user-generated assets such as listing images, page photos and floor plans on S3 (bucket com-gunnertech-jire). These assets are additionally CDNed via Cloudfront.
2. Serving static assets such as JS, CSS and graphic files from Cloudfront.

#### Property Base

Property Base is a part of Salesforce and is the CRM tool JIRE agents use to manage their workflow, including reports, prospects, clients and contacts.

Whenever a listing is saved via Refinery, the app uses the Salesforce API via Restforce to sync the corresponding Property and Listing record in Property Base.

Just to make that clear - PB splits listings into a Listing and Property object. A Property object in PB generally has multiple listings. Whereas in Refinery a property and listing are synonymous.

#### Barefoot

Barefoot is a third party which provides a CMS and management tools to JIRE for their Rental properties.

Stephanie uses these tools to create and manage rental listings, including photos, info and availability.

See the credentials section for login information.

Barefoot is simply used as a backend repository for management and provides a SOAP API, which Bluetent consumes, for building a corresponding front end to consumers/users.

Right now, there is no direct integration with this app and Barefoot

#### Bluetent

Bluetent is another third party as mentioned above.

Bluetent uses Drupal to consume Barefoot's API and render a UI to end consumers, which can be seen here: https://rentals.johnsislandrealestate.com/

The only direct integration this app has with Bluetent is via a link in the main navigation bar, linking users to the url above.

Bluetent is also providing consulting on best practices when using and integrating Campaign Monitor.

#### Luxury Real Estate

Luxury Real Estate, which you can access here http://www.luxuryrealestate.com/, is a Zillow for high-end homes.

This app sends all active listings in Refinery to Luxury Real Estate each night via an FTP push.

#### Zillow

The app generates a feed which Zillow and Trulia consume to create and update listings on their end.

All active listings are made available to Zillow at http://www.johnsislandrealestate.com/listings.xml

Zillow parses this URL periodically.

#### Campaign Monitor

Campaign Monitor is web-based contact software that JIRE uses to manage its newsletters, prospects, owners and renters.

Unlike Property Base, which also duplicates some of this functionality, Robyn manages campaign monitor and uses it for marketing purposes.

Right now, when a new user creates an account in the app, the app automatically adds them as a prospect to the campaign monitor list.

Additionally, there are other areas throughout the site (contact, inquiry, newsletter signup) that attempt to capture user information and preference (sales or rental) and subscribe them to the appropriate list.

We are also in the process of working through integrating this with Property Base so contacts are synced between the two.

## Contacts

Please refer to the following document for contacts related to this project

https://docs.google.com/spreadsheets/d/1UNM9BximBI7I2exC7NwNys7ZDqPSyvTJPrpkt2TM9NU/edit?usp=sharing

## Credentials

Credentials for this project are stored in this Google Doc (https://docs.google.com/spreadsheets/d/1o2OLPyjT3XO6b5NGj_8lrOPLBkpsPlUpjqC5LUvMy94/edit#gid=0)

Please request access from Cody if you can not access it.

## Workflow

Gunner Technology and JIRE meet once a week via teleconference for a 30-minute status meeting.

The first of these each month is used to plan the development work to be done for that month.

After that, each status meeting is used to review the plan and decide if it needs to be changed for more immediate issues.

We are currently discussing an expanded role for Gunner Technology, which would change our iteration length from a month to two weeks.

All our issues are created and tracked via GitHub.

## Our Responsibilities

1. Make sure all issues are ticketed in GitHub and updated in detail (this means translating emails into tickets)
2. Make sure issues are clear - one issue per issue
3. Maintain a clear tracking of issues - what's backlogged vs what's being worked on.
4. Respond to all emails within 24 hours either via a reply or issue creation/update

## Client Communications
Any JIRE representative should either use the @johnsisland GitHub account or their own GitHub account to communicate any issues that arise.

Client should not use email directly as this potentially will not reach everyone on the team. Creating an issue automatically *will* email everyone on the team.

Also, please do not label or assign the issue.

A person on the GT team will address the issue within 24 hours. Please note, that "address" does not necessarily mean fix.

Discussion will take place in the issue to determine if the issue is urgent enough to bump something out of the current iteration.

If the issue is determined to be urgent, a GT team member will move it into the current milestone and assign it to someone to do.

Otherwise, the GT team member will move it into the "Backlog" milestone, which means that it will be addressed during the next weekly meeting.
