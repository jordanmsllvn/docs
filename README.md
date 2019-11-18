# Calnetix 2019 Development

## Overview

The main areas of uncertainty fall into the following categories:

* Hosting environments
* Page redirects
* Editorial Workflows
* Form Building
* Performance
* Security

To give some context for the rest of the document, the basic plan is to split
the implementation into two servers:

* Drupal
  * Handles all content, workflow, routing, etc.
  * Basically anything that deals with the content, the structure of the content,
    the state of the content(publishing workflow etc), or the routing to the content.
* Node
  * Handles the server side rendering and presentation of React code.
  * Any external API integrations will also see huge benefits in terms of
    maintainability and performance by being put on this layer.
  * Complex data operations or business logic will be much more managable on
    this layer.

### Hosting Environments

We need:

So we have two main options here:

* Run a Drupal server and an additional Node.js server in parallel.
  * This gives us the most flexibility and allows us to offload server side
    business logic onto a Node.js server which by its asynchronous nature is
    much better for things like api integrations and complex operations.
  * Allows us to run node as a sort of 'bastion host' to our Drupal API, which
    can give us some seriously great security over traditional Drupal or
    publically exposed JSON:API.
* Run a Drupal server, then compile our entire frontend into a static site (with
  continuous automatic deployment when content changes are made) and
  host the static files on something like Netlify (or even s3 for super cheap
  and reliable hosting!)

While the second option is certainly 'cooler'. I think the first option is the
way to go due to the extra benefits we gain from a node.js server.

I should also point out that both of these options are interchangeable at any
time. Next.js supports server side rendering as well as static site export.

As far as actual hosting providers go, I assume we will want to use Acquia or
pantheon for the Drupal hosting. Its important to remember that we are
offloading all of the page rendering concerns to a much faster system, so the
hosting at this level likely isnt nearly as important as on a traditional Drupal
site. I think at the very least we just need managed database and continuous
deployment (with dev/stage/prod environments etc..)

It technically doesnt matter where the Node server is hosted. Acquia offers
node.js hosting, but honestly its pretty limited and likely extremely overpriced
compared to most competitors (They aren't very forthcoming with pricing so I am
unable to determine how much it costs to run an additional node server in
parallel)

Were likely going to want to go for a PaaS (Platform as a Service) for hosting
our node server. Something like Netlify or Heroku. This is where most of our
development will live so its important we choose one that keeps 'fiddling' with
devops and server managment to an absolute minimum.

These handle continous deployment slug/artifact deploys to production, staging
environments, PR review servers etc... They also handle SSL and all that for us.
Basically we want to use a PAAS so we never have to worry about dev ops, ever.  
Stuff works right out of the box with little to no configuration. Its literally
hook up to the git hub repository and go.

Heroku and Netlify are both decent options. Netlify starts at about $45 a month
for a production setup (however its free for development).

Heroku is a bit less feature rich with its bare bones servers, and a bit
cheaper. It also seems more geared to web applications then websites, and its
feature set reflects that. (for example, Netlify includes netlify edge which
makes caching and deploying the front end statically a breeze)

The comparison of the features sets and pricing can be found here:
[Heroku](https://www.heroku.com/pricing/)
[Netlify](https://www.netlify.com/pricing/)

Both are pretty fantastic platforms and I dont think we can go wrong. Also
migrating from one platform to another would be a few hours at most in case we
ever needed to change course.

### Page redirects

We need a way to handle ever changing paths from drupal while retaining redirect
unctionality.The best solution is in the form of an existing Drupal module
called "Decoupled Router"

#### Solution: Decoupled Router module

##### What it provides

This provides an endpoint that resolves information about an entity like what
type it is so our react app can display appropriately. This information doesn't
include the entity data itself. Only information describing what it is and where
to get the data. So there needs to be another roundtrip to the server to actually
receive the data we want.

##### The trade offs

Unfortunately so many roundtrips could cause performance issues, so we need to
use something called Subrequests to combine multiple requests into a single
round trip.

Drupal has a module to address this called 'Subrequests'.

GraphQL could be a better solution to this in the future depending on how its
implemented with Drupal, however I think on top of everything else it might
not be the best idea to go down that road for the first project. The subrequests
route is well documented and practiced already.

##### Proposed Implementation Details

Implementation is fairly simple.

The Node/React layer will always render a single react page no matter what the
route is. Anything past the initial '/' in the route will be forwarded to the
Decoupled Router module with attached subrequests to attach all the necessary
data for that route. The React page will look at the content type information
and data returned from  drupal for that route, and render in the correct modules
accordingly and populate them with content.

Once the page loads once, all new page loads will be done by swapping out components
and populating them with new content, essentially becoming an extremely fast single
page application. Node server side rendering will always ensure that search engines
crawl and index the site as if it were a plain old traditional website.

Here is an example of a complete url request to render flow:

1. End user hits a route `/products/widget-5000`
2. Node server receives the request. And prepares to render our single
   index.html. In doing so it calls our React Code.
3. React then takes the path `/products/widget-5000` and posts it to Drupal's decoupled
router. Along for the ride in this post are additional subrequests that instruct
Drupal to also grab the content for the node at that url, as well as any relations
we need.
4. Drupal returns a payload that includes what type of node lives at that url path,
as well as the content.
5. React looks at the payload, and sees that it's a product type node. It then
assembles a product page, and populates all the necessary components with their
respective content.
6. Node serves the HTML generated by React to the user's browser.
7. User then clicks a link to `/products/widget-10000`
8. The Process repeats for the new URL, but rather than node returning a new html
page when all is said and done, React re-renders the new page in place without
requesting any new html from the server. Possibly using some page transitions
effects if desired.

### Editorial Workflows

As far as I am able to tell. A piece of content is either published or not.
That is the only thing we care about as far as presenting to the end user.

Previews can work in much the same way they do already. If a user is logged in,
they should have access to certain unpublished revisions. They will be able
to preview this through a specific request to that revision that will by
nature only be allowed for authenticated users with permissions to that resource
(JSON:API defers completely to drupal's internal access control and user
management.).

Unfortunately, due to being in between two worlds(JSON:API Standard
Specification and Drupal), the json:api module has limited support
for revisions currently. However, it should currently be enough for
our needs

See: [Revisions in JSON:API](https://www.drupal.org/docs/8/modules/jsonapi/revisions)

Given the current state of headless drupal and JSON:API. It would be unwise
to attempt to enable "live/in-place" editing of content and revisions. While
this is certainly possible and can probably provide a fantastic editing
experience through real-time React-components, It will take a good amount of
planning and discovery to get the experience right. Also, Drupal isn't quiet
there yet in terms of api capability. Although it looks like it will be in
the near future. This might be an area we can provide some serious value to
back to the Drupal community and get Chronos' name out.

### Form building

Looks like the json api provides us with a yaml string defining
webforms that looks like so (pulling from the sitime contact form):

```text
  "your_role:
    '#type': select
    '#title': 'Your Role'
    '#wrapper_attributes':
      class:
        - 'form-item form-type-select'
    '#options':
      'Potential Customer': 'Potential Customer'
      'Current Customer': 'Current Customer'
      'Potential Partner': 'Potential Partner'
      'Current Partner': 'Current Partner'
      Analyst/Press: Analyst/Press
      Other: Other
    '#required': true
  first_name:
    '#type': textfield
    '#title': 'First Name'
    '#required': true
  etc...
```

We parse this into a form in react. Easy Peasy.

There is, however, a catch... JSON:API doesnt allow POSTING unless your user has
permissions. Which can get messy for anonymous users filling out contact forms
etc. This can be addressed by using a bastion host setup (which we should
eventually do either way for an extra layer of security).

However, the community accepted solution seems to be to use this module for
posting webform data back to Drupal:
[Webform REST](https://www.drupal.org/project/webform_rest)

Either way, we have a very clean solution here that should work great out of the
box.

### Localization / Translation

Content translations are exposed through JSON:API and should work out of the box.

The main consideration we have then is a good replacement for Drupal's T function.

Theres are TONS of great T function replacements. T functions are extremely
simple to implement, and techinically we could write our own that meets all our
requirements in under an hour. However, its best to use an alreadyu documented
system:

[react-i18next](https://github.com/i18next/react-i18next)

^^^ 323,827 installs a week. I think its safe to say React has us covered on
the translations front.

No real comprimises to be made on the translations front.

### Performance

#### Subrequests

The first line of defense against performance issues will be packaging up
multiple requests into a single request as subrequests using the drupal
subrequests module. As well as using
[includes](https://www.drupal.org/docs/8/modules/jsonapi/includes).
However, this will only remove latency from network requests, and may not do
much in terms of Drupal data-read speeds.

#### Caching

Drupal is not renowned for its performance, and can become a bottleneck to getting
extremely fast page loads and take full advantage of the new front-end systems.
Usually this is mitigated by copius amounts of caching.

While the front end systems are really fast and will mitigate the need
for caching, we may find that the roundtrips to get data from Drupal are slowing
our page loads down. Especially if we ever need to make multiple requests
for a single page load.

In traditional setups, a cache would go between the rendered html and the end
user. With the speed of Node+React we are unlikely to gain much benefit from
that approach until we start seeing HUGE amounts of concurrent use. With this
new system we are only ever putting strain on the server for rendering for the
very first page load. All the rest of the rendering is offloaded to the user's
browser which gives us a huge advantage in terms of server load.

We may find, however, that we dont need a cache for a production ready Calnetix
site, but either way we should be prepared for the possibility of needing some
caching.

### Security

Security is certainly a big concern when moving to new technology. Luckily, we
aren't actually using much new tech here on the backend. By essentially cutting
out a large portion of Drupal's front-facing functionality, we are actually
reducing attack surface area.

However, JSON:API does use Drupal's entity API, and we are essentially exposing
that API over public API endpoints. This could expose new surface area to
attacks. So I think it would be best if we restrict our uses of the api
to read-only, and disable all write operations. It would also be wise to only
expose as much of the API as we actually need to use, and disable the rest.

It might be worth removing public access to the JSON:API and essentially use
our node server as a 'bastion host' for our data. Assuming we sanitize requests
on the node server we essentially remove ourselves from drupal
security concerns all togther and make the site pretty damned untouchable.

## Proposed Development Plan

In order to minimize risk and make sure we keep on top of any possible problems
that arise and stay within budget, I propose a step by step plan for developing
this project:

### Step 1. Setup and Initial Entities

#### Tim

* Setup initial Drupal Development server on Aquia
* Install/enable needed decoupled modules. (JSON:API, Decoupled Router,
  Subrequests, Webform REST)
* Build out a set of entities needed for a select page/content-type that can
  provide a good example for the front end integration covering as many edge
  cases as possible. (Paragraphs, Image styles, Web Forms)

#### Jordan

* Build out a basic node server that can consume data from the drupal
  development server
* Setup continuous deployment on the chosen PaaS.
* Setup routing system and decoupled router integration.

### Step 2. Make Basic Working Decoupled Site

#### Tim

* Continue building out entities as desired

#### Jordan

* Build out a page based on the Drupal entities built in the last step
* Identify any issues that pop up in the integration and work with Tim to come
  up with solutions and resolve.

### Step 3. Prepare for rapid front-end development

#### Tim & Jordan (and possibly David?)

* Review the implementation on the front end and backend so far and address any
  concerns or hesitations.
* Define the general structure of the React-facing data contracts based on what
  was learned from the first page buildout.

#### Tim

* Continue to build out the backend.

#### Jordan

* Lock-in the development patterns for front-end based on waht was learned from
  the first page build out.
* Stub out the data contracts with demo data.

### Step 4. Frontend Push

#### Tim

* Finish up backend

#### Jordan

* Knock out all the React Components using the dummy contracts

### Step 5. Integration

#### Tim & Jordan

* Pipe real data into the data contracts with JSON:API queries.

### Step 6. Hardening

#### Tim & Jordan

* Identify what peices of the json api we arent using and disable them to
  increase security.
* Test performance and prepare additional caching if needed.


### Step 7. Content population, and Production server setup etc

