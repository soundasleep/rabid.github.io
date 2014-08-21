---
layout: post
title:  "Using omniauth-google-oauth2 for Google OAuth2 with Ruby on Rails"
date:   2014-08-21 17:21:58
author: jevon
---

I tried doing this <a href="http://nationbuilder.com/ruby_api_example">manually using the `oauth2` gem</a>, but while it <a href="https://github.com/soundasleep/rrw/commit/6f83482fe0d25fa05ddc24c9020b69afefddf0a2">kind-of works</a>, getting any sort of user information requires you to <a href="http://openid.net/specs/draft-jones-json-web-token-07.html">decode a JWT payload</a> (ugh, [[OpenID]] why does [[Google]] no longer support you).

Instead I got it working with just the `omniauth-google-oauth2` gem. Based on <a href="http://blog.myitcv.org.uk/2013/02/19/omniauth-google-oauth2-example.html">this tutorial</a> which is for an older version of Rails:

1. Add new bundles and run `bundle install`:

[code]
gem 'haml'
gem 'figaro'
gem 'omniauth-google-oauth2'
[/code]

2. Create a new controller for sessions, and a new database model for users:

[code]
rails generate session_migration
rails generate controller sessions
rails generate model User provider:string uid:string name:string refresh_token:string access_token:string expires:timestamp
rake db:migrate
[/code]

3. Configure the session to use a database rather than <a href="http://stackoverflow.com/questions/9473808/cookie-overflow-in-rails-application">cookies which have a 4KB limit</a>:

[code]
# config/initializers/session_store.rb

Example::Application.config.session_store :active_record_store
[/code]

4. Create your SessionsController:

[code ruby]
# /app/controllers/sessions_controller.rb

class SessionsController < ApplicationController
  def create
    auth = request.env["omniauth.auth"]
    user = User.where(:provider => auth["provider"], :uid => auth["uid"]).first_or_initialize(
      :refresh_token => auth["credentials"]["refresh_token"],
      :access_token => auth["credentials"]["token"],
      :expires => auth["credentials"]["expires_at"],
      :name => auth["info"]["name"],
    )
    url = session[:return_to] || root_path
    session[:return_to] = nil
    url = root_path if url.eql?('/logout')

    if user.save
      session[:user_id] = user.id
      notice = "Signed in!"
      logger.debug "URL to redirect to: #{url}"
      redirect_to url, :notice => notice
    else
      raise "Failed to login"
    end
  end

  def destroy
    session[:user_id] = nil
    redirect_to root_url, :notice => "Signed out!"
  end
end
[/code]

5. Login to your <a href="https://console.developers.google.com/project">Google Developers Console</a>, create a new Project, and visit *APIs & Auth*:

* *APIs:* enable Contacts API and Google+ API, to prevent <a href="http://stackoverflow.com/a/23904532/39531">Access Not Configured. Please use Google Developers Console to activate the API for your project.</a>
* *Consent screen:* make sure you have an email and product name specified
* *Credentials:* create a new Client ID of type web applicaton, setting your _Redirect URI_ to http://localhost:3000/auth/google_login/callback 

6. Edit `config/application.yml` with these Google keys (`figaro` will make all of these settings available through `ENV[property]`):

[code yml]
# config/application.yml

OAUTH_CLIENT_ID: "<CLIENT_ID>"
OAUTH_CLIENT_SECRET: "<CLIENT_SECRET>"
APPLICATION_CONFIG_SECRET_TOKEN: "<A LONG SECRET>"
[/code]

7. Enable Omniauth to use Googleauth2 as a provider (`approval_prompt` must be an empty string <a href="http://blog.myitcv.org.uk/2013/02/19/omniauth-google-oauth2-example.html">otherwise it will force a prompt on every login</a>):

[code ruby]
# /config/initializers/omniauth.rb

Rails.application.config.middleware.use OmniAuth::Builder do
  provider :google_oauth2,
    ENV['OAUTH_CLIENT_ID'],
    ENV['OAUTH_CLIENT_SECRET'],
    {name: "google_login", approval_prompt: ''}
end
[/code]

8. Enable new routes:

[code ruby]
# /config/routes.rb

Example::Application.routes.draw do
  # ...
  get "/auth/google_login/callback" => "sessions#create"
  get "/signout" => "sessions#destroy", :as => :signout
end
[/code]

9. Now finally, you can add login/logout links in your navigation template, or whatever:

[code haml]
# /app/views/home/index.html.haml

%p
  - if current_user
    = link_to "Sign out", signout_path
    = current_user.name
    = current_user.inspect
  - else
    = link_to "Sign in with Google", "/auth/google_login"
[/code]

10. Which uses a helper method, `current_user`, which loads the `User` model based on the session `user_id`:

[code ruby]
# /app/controllers/application_controller.rb

class ApplicationController < ActionController::Base
  // ...
  helper_method :current_user

  private

  def current_user
    @current_user ||= User.find(session[:user_id]) if session[:user_id]
  end
end
[/code]



A quirk of pytz, lead to learning something interesting about New Zealand.


    In [2]: import pytz; pytz.timezone("Pacific/Auckland")
    Out[2]: <DstTzInfo 'Pacific/Auckland' NZMT+11:30:00 STD>
    
    In [3]: import datetime; datetime.datetime(2011, 06, 12, 12, 54, 23, 00, pytz.timezone('Pacific/Auckland')).strftime('%Y-%m-%d %H:%M:%S %Z%z')
    Out[3]: '2011-06-12 12:54:23 NZMT+1130'

What's that? Our timezone is Auckland, with an offset of 11:30??

<!--break-->


That's because New Zealand hasn't always been in UTC+12  (or +13 in summer). We used to synchronise our clocks to the sun, 12pm noon being when the sun was over head -- but the advent of the telegraph lead to a country wide standardisation of time -- the winner was Christchurch, it being near the centre of NZ in longitude. Christchurch is 11 1/2 hours ahead of GMT, thus we standardised on GMT+11:30.

<a href="http://www.teara.govt.nz/en/timekeeping/page-2">There's lots of intesting details over at Te Ara</a>.

    In [4]: datetime.datetime(2011, 06, 12, 12, 54, 23, 00, pytz.timezone('NZ')).strftime('%Y-%m-%d %H:%M:%S %Z%z')
    Out[4]: '2011-06-12 12:54:23 NZMT+1130'

likewise for any other country, it uses the oldest timezone info it
can find (aka the first one).
Finland ditched HMT in the 1920s.

    In [11]: pytz.timezone("Europe/Helsinki")
    Out[11]: <DstTzInfo 'Europe/Helsinki' HMT+1:40:00 STD>


For Python, it's not what timezone NZ is, but what timezone NZ was using on that datetime. If I feed it 2011, it'll correct pick UTC+12.

So I have to call it like this:

    In [51]: pytz.timezone('Pacific/Auckland').localize(datetime(2011, 06, 03, 12, 54, 23, 00))
    Out[51]: datetime.datetime(2011, 6, 03, 12, 54, 23, tzinfo=<DstTzInfo 'Pacific/Auckland' NZST+12:00:00 STD>)

or something in summer

    In [52]: pytz.timezone('Pacific/Auckland').localize(datetime(2011, 02, 06, 12, 54, 23, 00))
    Out[52]: datetime.datetime(2011, 2, 6, 12, 54, 23, tzinfo=<DstTzInfo 'Pacific/Auckland' NZDT+13:00:00 DST>)

or something pre Tiriti O Waitangi:

    In [53]: pytz.timezone('Pacific/Auckland').localize(datetime(1829, 02, 06, 12, 54, 23, 00))
    Out[53]: datetime.datetime(1829, 2, 6, 12, 54, 23, tzinfo=<DstTzInfo 'Pacific/Auckland' NZMT+11:30:00 STD>)

To be clear, this isn't a bug -- this is me not reading the manual and making assumptions about calls -- but if I hadn't I would never have learned this trivia.

Enjoy.

