# Weather forecaster app with Ruby on Rails 


## Scope

1. Use Ruby On Rails. 

2. Accept an address as input. 

3. Retrieve forecast data for the given address. This should include, at minimum, the current temperature. Bonus points: retrieve high/low and/or extended forecast.

4. Display the requested forecast details to the user.

5. Cache the forecast details for 30 minutes for all subsequent requests by zip codes. Display indicator in result is pulled from cache.


## Set up Rails

This app is developed on a MacBook Pro M1 with macOS Ventura.


### Install asdf

I like to use the `asdf` version manager to install software such as programming languages, because `asdf` makes it easier for me to manage multiple versions, environment paths, and dependencies.

Install `asdf` version manager via `brew`:

```sh
% brew install asdf
% echo -e "\n. $(brew --prefix asdf)/asdf.sh" >> ~/.zshrc
% echo -e "\n. $(brew --prefix asdf)/etc/bash_completion.d/asdf.bash" >> ~/.zshrc
% source ~/.zshrc
```


### Install Ruby

I like to install `ruby` using the latest version, and via `brew` and `asdf`.

To do this on the MacBook Pro M1 with macOS Ventura, the installer requires the `capstone` package library files and include files.

Set up `capstone`:

```sh
% brew install capstone
% export LDFLAGS="-L"$(brew --prefix capstone)"/lib"
% export CPPFLAGS="-I"$(brew --prefix capstone)"/include"
```

Add the `asdf` plugin:

```sh
% asdf plugin add ruby
% asdf plugin-update ruby
```

Install Ruby and use it:

```sh
% asdf install ruby latest
% asdf global ruby latest
```


### Install Rails

Install Ruby on Rails:

```sh
% gem install rails
```


### Install Google Chrome

Install Google Chrome for Ruby on Rails system tests:

```sh
% brew install google-chrome
```


## Set up the app


### Create a new app

Create a new Ruby on Rails app and test it:

```sh
% rails new forecaster --skip-activerecord
% cd forecaster
% bin/rails test
% bin/rails test:system
% bin/rails server -d
% curl http://127.0.0.1:3000
% lsof -ti:3000 | xargs kill -9
```


### Add flash

I like to use Rails flash messages to show the user notices, alerts, and the like. I use some simple CSS to make the styling easy.

Add flash messages that are rendered via a view partial:

```sh
% mkdir app/views/shared
```

Create `app/views/shared/_flash.html.erb`:

```ruby
<% flash.each do |type, message| %>
  <div class="flash flash-<% type %>">
    <%= message %>
  </div>
<% end %>
```


## Accept an address as input

We want a controller can accept an address as an input parameter. 

A simple way to test this is by saving the address in the session.


### Add faker gem

To create test data, we can use the `faker` gem, which can create fake addresses.

Edit `Gemfile` and its `test` section to add the `faker` gem:

```ruby
gem "faker"
```

Run:

```sh
bundle
```


### Generate forecasts controller

Generate a forecasts controller and its tests:

```sh
% bin/rails generate controller forecasts show
```

Write a test in `test/controllers/forecasts_controller_test.rb`:

```ruby
require "test_helper"

class ForecastControllerTest < ActionDispatch::IntegrationTest

  test "show with an input address" do
    address = Faker::Address.full_address
    get forecasts_show_url, params: { address: address }
    assert_response :success
    assert_equal address, session[:address]
  end

end
```

Generate a system test that will launch the web page, and provide the correct placeholder for certain future work:

```
% bin/rails generate system_test forecasts
```

Write a test in `test/system/forecasts_test.rb`:

```ruby
require "application_system_test_case"

class ForecastsTest < ApplicationSystemTestCase

  test "show" do
    address = Faker::Address.full_address
    visit url_for \
      controller: "forecasts", 
      action: "show", 
      params: { 
        address: address 
      }
    assert_selector "h1", text: "Forecasts#show"
  end

end
```

TDD should fail:

```sh
% bin/rails test:all
```

Implement in `app/controllers/forecasts_controller.rb`:


```ruby
class ForecastsController < ApplicationController

  def show
    session[:address] = params[:address]
  end

end
```

TDD should succeed:

```sh
% bin/rails test:all
```


### Set the root path route

Edit `config/routes.rb`:

```ruby
# Defines the root path route ("/")
root "forecasts#show"
```



## Get forecast data for the given address

There are many ways we could get forecast data. 

* We choose to convert the address to a latitude and longitude, by using the geocoder gem and the ESRI ArcGIS API available [here](https://developers.arcgis.com/sign-up/)

* We choose to send the latitude and longitude to the OpenWeatherMap API available [here](https://openweathermap.com)

* We choose to implement each API as an application service, by creating a plain old Ruby object (PORO) in the directory `app/services`

Run:

```sh
% mkdir -p {app,test}/services
% touch {app,test}/services/.keep
```


### Set ArcGIS API credentials

Edit Rails credentials:

```sh
EDITOR="code --wait"  bin/rails credentials:edit
```

Add your ArcGIS credentials by replacing these fake credentials with your real credentials:

```ruby
arcgis_api_user_id: alice
arcgis_api_secret_key: 6d9ecd1c-2b00-4a0e-89d7-8f250418a9c4
```


### Add Geocoder gem

Ruby has an excellent way to access the ArcGIS API, by using the Geocoder gem, and configuring it for the ArcGIS API.

Edit `Gemfile` to add:

```ruby
# Look up a map address and convert it to latitude, longitude, etc.
gem "geocoder"
```

Run:

```sh
bundle
```


### Configure Geocoder

Create `config/initializers/geocoder.rb`:

```ruby
Geocoder.configure(
    esri: {
        api_key: [
            Rails.application.credentials.arcgis_api_user_id, 
            Rails.application.credentials.arcgis_api_secret_key,
        ], 
        for_storage: true
    }
)
```


### Create GeocodeService

We want to create a geocode service that converts from an address string into a latitude, longitude, country code, and postal code.

Create `test/services/geocode_service_test`:

```ruby
require 'test_helper'

class GeocodeServiceTest < ActiveSupport::TestCase

  test "call with known address" do
    address = "1 Infinite Loop, Cupertino, California"
    geocode = GeocodeService.call(address)
    assert_in_delta 37.33, geocode.latitude, 0.1
    assert_in_delta -122.03, geocode.longitude, 0.1
    assert_equal "us", geocode.country_code
    assert_equal "95014", geocode.postal_code
  end

end
```

Create `app/services/geocode_service`:

```ruby
class GeocodeService 

  def self.call(address)
    response = Geocoder.search(address)
    response or raise IOError.new "Geocoder error"
    response.length > 0 or raise IOError.new "Geocoder is empty: #{response}"
    data = response.first.data
    data or raise IOError.new "Geocoder data error"
    data["lat"] or raise IOError.new "Geocoder latitude is missing"
    data["lon"] or raise IOError.new "Geocoder longitude is missing"
    data["address"] or raise IOError.new "Geocoder address is missing" 
    data["address"]["country_code"] or raise IOError.new "Geocoder country code is missing"
    data["address"]["postcode"] or raise IOError.new "Geocoder postal code is missing" 
    geocode = OpenStruct.new
    geocode.latitude = data["lat"].to_f
    geocode.longitude = data["lon"].to_f
    geocode.country_code = data["address"]["country_code"]
    geocode.postal_code = data["address"]["postcode"]
    geocode
  end

end
```
