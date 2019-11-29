# Rails 6, Webpacker and the GOVUK Frontend
We love [Ruby on Rails](https://rubyonrails.org/) and the GOVUK Frontend (the implementation of the [GOVUK Design system](https://design-system.service.gov.uk/)) here at dxw and have built lots of services with them.

Every service I have worked on involved both and it's not always been smooth sailing to get them to play nicely together!

With the release of Rails 6, I was keen to see whether using [Webpacker](https://github.com/rails/webpacker) (enabled by default in Rails 6) would make things any smoother.

Webpacker is '[Webpack](https://webpack.js.org/) the Rails way'. Which to me, means a clever, opinionated and sensible default setup of Webpack integrated into Rails. There was a nice intro to Webpacker at [RailsConf 2019 that is worth a watch](https://youtu.be/2v4ySqyua1s)

I thought it would be interesting to share my experience of getting the GOVUK Frontend into Rails 6 using Webpacker. Bear in mind, this is neither the definitive way to achieve this goal or a full blown tutorial!

Let's get a fresh Rails project, we'll skip some of the components we won't need, like [Sprockets](https://github.com/rails/sprockets-rails), we'll call it 'rails-and-govuk-frontend':

`rails new --skip-sprockets --skip-action-cable --skip-active-storage rails-and-govuk-frontend`

Inside our new project, let's get a controller and a view to work with:

`rails generate controller Home index --no-assets --no-helper`

And set the view to the root of the site by updating the routes file:

`config/routes.rb`
`root 'home#index'`

Next we want the GOVUK page template as our application layout, we'll use the [template from here](https://design-system.service.gov.uk/styles/page-template/) and include a couple of Rails parts:

`app/views/layout/application.html.erb`

We'll add some css classes in our view so we can see when things are working:

`app/views/home/index.html.erb`

We also want the GOVUK Frontend package, which we get with yarn:

`yarn add govuk-frontend`

Running the application now we get this:

Hmmm, not a great looking service, but it is the basics we need, now we need to wire things up!

Webpacker in Rails uses the idea of 'packs'. Our aim is to make a pack that contains all the things we need from the GOVUK Frontend: the  styles, JavaScript, images, fonts, etc.
These packs live in the new `javascript` directory inside our Rails project. We'll call our pack `govuk`:

`app/javascript/packs/govuk.js`

We also need some control over how the GOVUK Frontend styles will get compiled. To do this we'll make a [Sass](https://sass-lang.com/) file which sets things up and then loads all the GOVUK styles.

In this brave new Webpacker world all the assets are loaded via JavaScript (via Webpack), but don't panic, only in the build process, our service will still work for users without JavaScript. Because of this we keep all the components of our pack together in the new `javascript/packs` directory.

I know it's a little odd to store assets in the `javascripts` directory but we are following [convention over configuration](https://rubyonrails.org/doctrine/#convention-over-configuration) here!

Let’s create our Sass file:

`app/javascipts/packs/stylesheets/govuk.scss`

In this file, we'll set a variable the GOVUK styles uses internally and then import them like so:

````
$govuk-assets-path: "~govuk-frontend/govuk/assets/";
@import "govuk-frontend/govuk/all";
````

The `$govuk-assets-path` tells Sass where to load the images and fonts that are used in the styles from `govuk-frontend/govuk/assets/`, the `~` tells Sass to look in the `node_modules` directory and internally the GOVUK Sass will add either `images` or `fonts` to set the paths correctly.

Next a standard `@import` brings all the GOVUK styles into our file. Here `all` contains every GOVUK component, but we could load only the components we need at this point if we wanted even finer control.

There are also other options we could set, see the [GOVUK Frontend settings](https://github.com/alphagov/govuk-frontend/tree/master/src/govuk/settings) for more info.

Now we need to load the GOVUK JavaScript, import our Sass file and add any assets that are not loaded from the styles, which include things like the favicon, Open Graph image and more.
To do this we require them into our pack, remember everything is loaded via JavaScript so it goes into:

`app/javascipt/packs/govuk.js`

````
require("govuk-frontend/govuk/all").initAll()
require.context('govuk-frontend/govuk/assets/images', true)
import "./stylesheets/govuk.scss"
````

The first line grabs the GOVUK JavaScript and calls `initAll()` on it, which is a function in the GOVUK Frontend that initialises all the components on the page.

`require.context` is a [Webpack method](https://webpack.js.org/api/module-methods/#requirecontext) to add a whole directory of dependencies which we use here to load all the other images from the GOVUK Frontend package.

Lastly we import the stylesheet we just created.

Out application now has two packs 'application' and 'govuk', but it's not using them, let's change that…

We use the [`javascript_pack_tag`](https://www.rubydoc.info/github/rails/webpacker/Webpacker%2FHelper:javascript_pack_tag) helper to load the JavaScript packs into our application layout.

The application pack can go in the `<head></head>`, but he GOVUK JavaScript (and so our pack) must be [loaded at the end of the document](https://github.com/alphagov/govuk-frontend/blob/master/docs/installation/installing-with-npm.md#option-1-include-javascript) so we add that before the closing body tag.

We also use the `stylesheet_pack_tag` helper to load the styles from our `govuk` pack back in the `<head></head>` again.

[app/views/layouts/application.html.erb]()

````
<%= stylesheet_pack_tag 'govuk', 'data-turbolinks-track': 'reload' %>
<%= javascript_pack_tag 'application', 'data-turbolinks-track': 'reload' %>
…
<%= javascript_pack_tag 'govuk', 'data-turbolinks-track': 'reload' %>
</body>
````

See the complete file:

[app/views/layouts/application.html.erb](https://github.com/dxw/blog-rails-webpacker-govk-frontend/commit/01dd907da0d959001c16a8e10c27d976e5fe62a1)

Now we have everything loaded we just need to update the static assets in our layout, there are quite a few, but essentially any url to an asset should be replaced with:

`<%= asset_pack_url('media/images/filename-of-the-asset') %>`

The 'media' part of the url is really important as it is added by Webpacker before each asset path. Once we bundle our packs, looking in `public` at the Rails project root you should see `public/packs/media/images` and `public/packs/media/fonts` populated with the assets.

Here's the complete layout with static asset paths:

[app/views/layout/application.html.erb]()

Let's see where we are, we need to run web packer to bundle all the assets into the pack:

`bin/webpack`

Or we can automatically bundle every time there is a change with:

`bin/webpack-dev-server`

(There is more to the webpack-dev-server than this, but that's another story.)

And running the application now we get this:

Looks great, but there is a problem that isn't obvious, all the styles are loaded into our JavaScript so users without JavaScript cannot see them! :(

To prove it here is our service with JavaScript disabled:

We should always consider styles and JavaScript [progressive enhancement](https://www.gov.uk/service-manual/technology/using-progressive-enhancement) so let's get this fixed.

To get the styles as a separate file that doesn't need JavaScript to load it we can change the default behaviour in Webpacker:

[config/webpacker.yml]()

`extract_css: true`

After restarting `webpack-dev-server` or running `webpack` and re-starting our Rails server, our styles come as a separate `css` file and load without JavaScript, we had to restart Rails because we made a configuration change.

By default Webpacker only uses this 'styles in JavaScript' approach in the development and test environments, so production, or our real live service, would have been okay, but I feel like I want all three environments to work the same way regardless.

And that is it, I am pretty happy to see that the GOVUK Frontend can be loaded with Webpacker without too much effort, once I got the idea of the packs I think it's a little simpler than using the asset pipeline. The approach I use with the asset pipeline means listing all the GOVUK static assets out which feels a little bit brittle to changes.

I like that this also makes managing the dependency the our application has on the GOVUK Frontend super easy, we can just use yarn to update the package or lock it to a specific version for example.

I hope you enjoyed our little adventure in Rails, Webpacker and the GOVUK Frontend! If you have questions or comments, please get in touch!
