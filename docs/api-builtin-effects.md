# Builtin Effects

re-frame supplies a small number of built-in effects which contribute to the API.

## What Are Effects? 

Event handlers, such as those registered using `reg-event-fx`, compute and return a map of effects which might look like this: 
```clj 
{:db  new-db
 :fx  [ [:dispatch [:some-id]]
        [:full-screen true]
        [:http     {:method :GET  :url "http://somewhere.com/"}]]}
```
That's a map with two keys:  `:db` and `:fx`.  Which, in turn, means there are two effects. There could be others.

An effect consists of an `id` and a `payload`. The `id` identifies the effect (so we know how to action it) and the `payload` 
carries additional information which will parameterise the action taken. 

So, if an effect map was `#!clj {:db m}`, it would contain only one effect with an `id` of `:db` and a payload of `m` (the value of `m`).

In the example above, you'll notice the other effect with an `id` of `:fx` has a vector payload. That vector is a sequenceof other effects, each with an `id` and a `payload`. So `:fx` is an effect which sequences the actioning of other effects. 

Certain of these effects are "builtin", such as `:db`, `:fx` or `:dispatch`. Others, like `:http` might come from a third-party library. This page lists the built-in ones.

## <a name="db"></a> :db

`reset!` `app-db` to be a new value. The associated `value` is expected to be a map. 

The `:db` effect has a special status. It will always be actioned before others. (Prior to v1.1.0 this guarantee did not exist. There were no ordering guarantees).

usage:
```clojure
{:db  some-map}   
```

In the wild, real usage might look like this: 
```clojure
(reg-event-fx
  :token 
  (fn [{:keys [db]} event]
    {:db  (assoc db :some-key some-val)}))     ;; <-- new value computed
```

## <a name="fx"></a> :fx

An effect which actions other effects, sequentially.

Expects a value which is a sequence, typically a vector.
Each element in the sequence represents one effect. 
Each element is a 2-tuple of (1) an effect id and (2) the payload of the effect (the value ultimately given to the registered effect handler as an argument). 

For example:
```clj
{:db  new-db 
 :fx  [ [:dispatch   [:some-id "extra"]]
        [:http-xhrio {:method :GET  :url "http://somewhere.com/"}]
        (when (> 2 3) [:full-screen true])]}
```

Notice the use of `when` to conditionally include or exclude an effect. Any `nil` found in a `:fx` sequence will be ignored. 

## <a name="dispatch"></a> :dispatch

`dispatch` one event. Expects a single vector.

usage:
```clojure
{:fx [[:dispatch [:event-id "param1" :param2]]] }
```

To dispatch multiple events:
```clojure
{:fx [[:dispatch [:event1 "param1" :param2]]
      [:dispatch [:second]]}
```
Effects in `:fx` are actioned in order, so the dispatched events will be queued and, later handled, in order supplied. FIFO.

## <a name="dispatch-later"></a> :dispatch-later

`dispatch` one or more events after a given delay. Expects a payload which is a 
map with two keys `:ms` (milliseconds) and `:dispatch` (the event).

usage:
```clojure
{:fx [ [:dispatch-later {:ms 200 :dispatch [:event-id1 "param"]}]
       [:dispatch-later {:ms 100 :dispatch [:event-id2 "param"]}]]}
```

Prior to re-frame v1.1.1 `:dispatch-later` required a seq of maps, since v1.1.1 it 
can also accept a single map. 
   
## <a name="deregister-event-handler"></a> :deregister-event-handler

Removes a previously registered event handler. Expects the event id for a previously registered event handler. 

usage:
```clojure
{:db new-db
 :fx [[:deregister-event-handler :my-id]])}
```


## <a name="dispatch-n"></a> :dispatch-n

> From v1.1.0 onwards, this effect is **deprecated** in favour of using `:fx` with multiple `:dispatch` tuples.

`dispatch` more than one event. Expects a seq of event vectors (typically a list of them). 

usage:
```clojure
{:db new-db
 :fx [[:dispatch-n (list [:do :all] [:three :of] [:these])]]}
```
Notes:

  1. The events will be dispatched in the order provided. And, because events are handled FIFO, the events will subsequently be processed in the order provided.
  2. nils in the event collection are ignored which means events can be added
conditionally:

     ```clojure
     {:db new-db
      :fx [[:dispatch-n (list (when (> 3 5) [:conditioned-out])
                           [:another-one])]]}
     ```