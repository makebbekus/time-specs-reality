# Time, Specs, and Reality
We want tests that are easier to understand and thus easier to fix and refactor. Reality is easy to understand.

![](http://upload.wikimedia.org/wikipedia/en/d/dd/The_Persistence_of_Memory.jpg)

## Data that mirrors reality

  Pickups have a **4 minute pickup window** and a cutoff **1 minute in the future**!?

```coffeescript
Factory.define 'pickup', Pickup, ->
    status: 'active'
    location: ->
      name: "#{Faker.Address.city()} Ironworks"
      address: Faker.Address.streetAddress()
      address2: Faker.Address.secondaryAddress()
      city: Faker.Address.city()
      state: Faker.Address.usState()
      zip: Faker.Address.zipCode()
      vagueAddress: '1st & Main St, San Francisco CA 94110'
      lat: Math.random() * 10
      lon: Math.random() * 10
      foodshed: 'sfbay'
      zone: 'san_francisco'
    pickupWindow: ->
      # vevents do not support ms resolution
      startAt: stripMs(clock.now() + 6.minutes())
      endAt: stripMs(clock.now() + 10.minutes())
    processCutoffOffset: - 1.minute()
    tzid: clock.pacific.tzid
```

FixedPickup has a realistic window but it's 2012 and the **date is buried in the factory**.
```coffeescript
Factory.define 'fixedPickup', Pickup, (args) ->
    actors =
      status: 'active'
      location:
        name: "#{Faker.Address.city()} Ironworks"
        address: '101 Main St'
        city: 'San Francisco'
        state: 'CA'
        zip: '94110'
        vagueAddress: '1st & Main St, San Francisco CA 94110'
        lat: Math.random() * 10
        lon: Math.random() * 10
        foodshed: 'sfbay'
        zone: 'san_francisco'
      pickupWindow:
        # vevents do not support ms resolution
        startAt: clock.pacific '2012-02-02 17:00'
        endAt: clock.pacific '2012-02-02 19:00'
      processCutoffOffset: -2.days()
      tzid: clock.pacific.tzid
      sharedDetails: {}
```

How about Foodubs that are open Monday through Friday and Vendors that have pickups with normal hours?
```coffeescript
  Factory.define 'foodhub', Foodhub, (args) ->
    orderCutoff = ->
      customized: true
      days: 1
      timeOfDay: "00:00"

    actors =
      name: "#{Faker.Address.city()} Foodhub"
      location:
        name: "#{Faker.Address.city()} Ironworks"
        address: Faker.Address.streetAddress()
        address2: Faker.Address.secondaryAddress()
        city: Faker.Address.city()
        state: Faker.Address.usState()
        zip: Faker.Address.zipCode()
        foodshed: 'sfbay'
      dropoffWindow: ->
        startAt: stripMs(clock.now() + 1.hours())
        endAt: stripMs(clock.now() + 3.hours())
      tzid: clock.pacific.tzid
      messages:
        dropoffInstructions: 'Orders can also be dropped off 1pm - 6pm the day before.'
      schedule:
        mo: {status: 'active', orderCutoff: orderCutoff()}
        tu: {status: 'active', orderCutoff: orderCutoff()}
        we: {status: 'active', orderCutoff: orderCutoff()}
        th: {status: 'active', orderCutoff: orderCutoff()}
        fr: {status: 'active', orderCutoff: orderCutoff()}
        sa: {status: 'inactive', orderCutoff: orderCutoff()}
        su: {status: 'inactive', orderCutoff: orderCutoff()}
        exceptionalDays: []

    # merge in argument paths
    for path, value of args
      _(actors).path path, value

    return actors
```

###Singleton Foodhubs
To simplify specs we have singleton foodhubs in the specs for `Factory.foodhubs.sfbay` and `Factory.foodhubs.nyc`

###Clock.now() same on server and client
`window.serverNow` embedded in client

## Stubbing Clock.now()
Market specs are easier to understand when you know what today is.

```coffeescript
require './helper'
emailer = require 'app/lib/emailer'
payments = require 'app/lib/payments'

describe 'Webstand Tests', ->
  {products, product, vendor} = {}

  beforeEach ->
    monday = clock.pacific '2014-05-05 06:00'
    spyOn(clock, 'now').andReturn monday

  describe 'Entry URLs', ->

    beforeEach ->
      {vendor} = Factory.create 'vendorWithActiveDays', activeDays: ['WE']
      {foodhub, products} = Factory.create 'fixedMarket', {vendor}

      expect(foodhub.getShoppableDays clock.now()).toEqual ['2014-05-07','2014-05-08','2014-05-09','2014-05-12','2014-05-13']
      expect(foodhub.firstShoppableDayAt clock.now(), {vendor}).toEqual('2014-05-07')


    describe 'product detail', ->
      thisWednesday = '2014-05-07'

      it 'respects pickupDay', ->
        @expectWebstandIsOnPickupDay(tryFor: thisWednesday, get: thisWednesday)

      it 'picks the next shoppable day if pickupDay is in the past (not shoppable)', ->
        yesterday = "2014-05-04"
        @expectWebstandIsOnPickupDay(tryFor: yesterday, get: thisWednesday)

      it 'picks the next shoppable day if pickupDay is too far in the future (not shoppable)', ->
        monthAhead = "2014-06-05"
        @expectWebstandIsOnPickupDay(tryFor: monthAhead, get: thisWednesday)

      it 'picks the next shoppable day if pickupDay is invalid (not shoppable for that vendor)', ->
        thisThursday = "2014-05-08"
        @expectWebstandIsOnPickupDay(tryFor: thisThursday, get: thisWednesday)

      it 'picks the next shoppable day if pickupDay is malformed', ->
        @expectWebstandIsOnPickupDay(tryFor: 'garbage', get: thisWednesday)

```
```coffeescript
  beforeEach ->
    @expectWebstandIsOnPickupDay = ({tryFor, get}) ->
      @browser.sync.navigateTo "/#{vendor.get('slug')}/products/#{products[0].get('_id')}?pickupDay=#{tryFor}"
      @browser.sync.waitFor -> $('.product-carousel-view-modal').is ':visible'
      expect(@browser.sync.evaluate -> $('.product-carousel-view-modal').text()).toContain products[0].get 'name'
      expect(@browser.sync.evaluate(-> $('select.fulfillment-day-chooser').val())).toBe get
```
