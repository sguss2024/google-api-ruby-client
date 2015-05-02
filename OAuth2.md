# Overview #
[OAuth 2.0](http://oauth.net/2/) is an emerging standard for accessing protected resources on the web. Google APIs and the google-api-ruby-client library support OAuth 2.0.

## Further Reading ##
  * For general information about OAuth 2.0, see the [OAuth 2.0 draft specification](http://tools.ietf.org/html/draft-ietf-oauth-v2).
  * For information about using OAuth 2.0 in various flows with Google APIs, see [Using OAuth 2.0 to Access Google APIs](https://developers.google.com/accounts/docs/OAuth2).

# Sample #
Here is a simple example using the ruby client and OAuth 2.0 to access the calendar API.

The full source code of this sample can be found [here](http://code.google.com/p/google-api-ruby-client/source/browse/calendar/calendar.rb?repo=samples).

```
rrequire 'rubygems'
require 'google/api_client'
require 'sinatra'
require 'google/api_client'
require 'logger'

enable :sessions

def logger; settings.logger end

def api_client; settings.api_client; end

def calendar_api; settings.calendar; end

def user_credentials
  # Build a per-request oauth credential based on token stored in session
  # which allows us to use a shared API client.
  @authorization ||= (
    auth = api_client.authorization.dup
    auth.redirect_uri = to('/oauth2callback')
    auth.update_token!(session)
    auth
  )
end

configure do
  log_file = File.open('calendar.log', 'a+')
  log_file.sync = true
  logger = Logger.new(log_file)
  logger.level = Logger::DEBUG
  
  client = Google::APIClient.new
  client.authorization.client_id = '...'
  client.authorization.client_secret = '...'
  client.authorization.scope = 'https://www.googleapis.com/auth/calendar'

  calendar = client.discovered_api('calendar', 'v3')

  set :logger, logger
  set :api_client, client
  set :calendar, calendar
end

before do
  # Ensure user has authorized the app
  unless user_credentials.access_token || request.path_info =~ /^\/oauth2/
    redirect to('/oauth2authorize')
  end
end

after do
  # Serialize the access/refresh token to the session
  session[:access_token] = user_credentials.access_token
  session[:refresh_token] = user_credentials.refresh_token
  session[:expires_in] = user_credentials.expires_in
  session[:issued_at] = user_credentials.issued_at
end

get '/oauth2authorize' do
  # Request authorization
  redirect user_credentials.authorization_uri.to_s, 303
end

get '/oauth2callback' do
  # Exchange token
  user_credentials.code = params[:code] if params[:code]
  user_credentials.fetch_access_token!
  redirect to('/')
end

get '/' do
  # Fetch list of events on the user's default calandar
  result = api_client.execute(:api_method => settings.calendar.events.list,
                              :parameters => {'calendarId' => 'primary'},
                              :authorization => user_credentials)
  [result.status, {'Content-Type' => 'application/json'}, result.data.to_json]
end
```