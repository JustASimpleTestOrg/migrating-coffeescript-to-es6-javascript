## 1 Wasting Coding Time
If you're a developer, you're going to empathize with my mood today. I spent 3 hours rewriting the old airpair.com API from CoffeeScript to ES6 JavaScript... and failed.

We all become Angry Cat every now and then. Its worse if your coding for yourself, rather than on someone else's dime. And more frustrating when you've promised a feature to your users and you know you're going to be late.

Today I was too stubborn to AirPair. I know why, I've had this feeling before. You get to a mind state where you decide, *"I just need to figure it out ON MY OWN so I don't feel stupid!"*

3 hours later, I feel stupid for not getting help. I blame it partly on my pride and partly on the limitations of the way AirPair currently works. I'm even more determined now to create the interaction I needed today to make it so easy to AirPair, I will do it every time I even get close to today's mind state.

In the meantime, here's what I figured out on my own re-writing Coffee Script as Harmony JS.

## 2 Migrating CoffeeScript to ES6

Here's some of the CoffeeScript I was to migrating. I got a lot of mileage from some class inheritance and heavy reliance on CoffeeScript fat arrows `=>`. Fat arrows in Coffee preserve `this` within the different functions of a class. I miss them SO MUCH! 

<!--?prettify lang=coffeescript linenums=true?-->    

    class BaseApi
      
      logging: off
      
      # Constructor for creating business logic service 
      Svc: -> $log "override in child class"

      # Read routes handled by this Api instance
      constructor: (app) ->
        @routes app

      # Override routers in each child class
      routes: (app) ->

      # Handle results consistently and DRY
      cSend: (res, next) ->
        (e, r) =>
          if @logging then $log 'cSend', e, r
          if e && e.status then return res.send(400, e)
          if e then return next e
          res.send r

      # AirPair Api middleware
      ap: (req, res, next) =>
        # Instantiate business layer service with user context
        @svc = new @Svc req.user

        # Create the callback to send the response
        @cbSend = @cSend res, next
    
        # dump info about the request if logging is on
        if @logging
          console.log req.url, req.body

        # finally, call next callback in the middleware chain
        next()

      # Default http operations
      create: (req) => @svc.create req.body, @cbSend
      list: (req) => @svc.getAll @cbSend
      detail: (req) => @svc.getById req.params.id, @cbSend



<!--?prettify lang=coffeescript linenums=true?-->    

    class WorkshopsApi extends require('./_api')

      Svc: require './../services/workshops'

      routes: (app) ->
        app.post  "/adm/workshop", @ap, @create
        app.get  "/adm/workshop", @ap, @list
        app.get  "/adm/workshop/:id", @ap, @detail



## 3 CoffeeScript to ES6 Tips

The following lessons are by no means an exhaustive or perfected discussion. Just what I learned mucking around today and where my knowledge is at.

### 3.1 Arrow Functions

ES6 JavaScript has no thin arrow function `->`. Essentially, CoffeeScript `->` == ES6 `=>`. There is no equivalent CoffeScript fat arrow function, and if you got lazy like me, you're going to have to change a lot if you're coming back from the dark side.

![Dark Side Cat](//www.starwarscats.com/wp-content/uploads/2013/01/darth-vader-cat.jpg) 

The other gotchas, that you'll get used to in a couple of days are (1) needing braces around multi-line functions (e.g. line 2 and 10 + line 4 and 9), and (2) remembering to use the `return` keyword.

<!--?prettify lang=javascript linenums=true?-->   
    
    cbSend(fn) {
      return (req, res, next) => {
        $log('inner.req', req)
        fn( (e , r) => {
          if (logging) { $log('cbSend', e, r) }
          if (e && e.status) { return res.send(400, e) }
          if (e) { return next(e) }
          res.send(r)
        })
      }
    }

You also might get initially tripped up on the function syntax in-side of classes.

<!--?prettify lang=javascript linenums=true?-->   

    // normal es6 function
    cbSend = (fn) => {
      ...
    }
    
    // function definition in a class
    cbSend(fn) {
      ...
    }

### 3.2 ES6 Destructuring Assignment

ES6 Destructuring Assignment has a bit more going on than CoffeeScript Descructuring Assignment. For example you can use it to Destructure arrays. Largely you can use it almost the same. Though I found two Gotchas today. (1) you need to start using the var keyword.

<!--?prettify lang=javascript linenums=true?-->   

    // Doesn't work
    {body,params} = req
    
    // Works
    var {body,params} = req

(2) ES6 Destructuring Assignment returns `undefined` when attributes on an object don't exist, where CoffeScript Destructuring Assignment returns `null`


### 3.3 ES6 Classes

This is as far as I got with converting my classes. I'm seriously wondering if I need to embrace functional programming and move away from Object inheritance after all these years.

I've gone ahead and commented the differences in-line. But the main limitation is no longer being able to think of everything inside your class sharing the same `this` context. It's a really bit deal, be prepared!

<!--?prettify lang=javascript linenums=true?-->   

    // Put logging on outer scope because 'this' isn't necessarily
    // available from one class function to the next
    var logging = false

    class BaseApi {
      
      constructor(app) {
        this.routes(app)
      }

      routes(app) { $log('override in child class') }

      // A bit more complicated than coffee version because we
      // Can't cheat with this context
      cbSend(fn) {
        return (req, res, next) => {
          fn( (e , r) => {
            if (logging) { $log('cbSend', e, r) }
            if (e && e.status) { return res.send(400, e) }
            if (e) { return next(e) }
            res.send(r)
          })
        }
      }

    }

<!--?prettify lang=javascript linenums=true?-->   

    import WorkshopsService from '../services/workshops'

    class WorkshopsApi extends BaseApi {

      // Need to define child constructor and call super because
      // we don't have class attributes like ln 3 in Coffee Version
      constructor(app) {
        var user = null // haven't yet figured out how to get user off req
        this.Svc = new WorkshopsService(user)
        super(app)
      }

      routes(app) {
        app.get( '/workshops/', this.list(this) )
        app.get( '/workshops/:slug', this.detail(this) )
      }

      //-- We have to pass self, as 'this' is taken from middleware chain
      list(self) {
        return self.cbSend( self.svc.getAll )
      }

      //-- Didn't figure this a nice was to do detail yet :{}
      detail(self) {
        return (req, res, next) => {
          var slug = req.params.slug
          self.svc.getBySlug(slug, (e , r) => {
            if (e && e.status) { return res.send(400, e) }
            if (e) { return next(e) }
            res.send(r)
          })
        }    
      }
    }


### 4 Summary

To be honest, I don't know if I'm missing a few bits of knowledge to make this awesome, or if what I'm trying to do was a waste of time from the start. I probably should have just build this whole piece with an AirPair and saved my Dark Kat coming out.