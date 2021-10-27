#### Naming

*   Modules which will be embed from **JS** should be named `JS.elm`. They should be stored in `ModuleName/Js.elm`. This module should expose only `main` function which should be a type of `Html` or `Program`.
*   Modules with complex logic should be split into two modules `ModuleName.elm` and `ModuleName/Internal.elm`. Basically we want to keep the implementation of the module secret. Thus we need to keep public API as small as possible. `ModuleName.elm` would expose only public functions and values so that module could be reused. And `ModuleName/Internal.elm` would expose everything needed for tests. `ModuleName/Internal.elm` could be imported only by `ModuleName.elm`, otherwise the build on Jenkins will fail.

#### Code style

**Use descriptive names instead of tacking on underscores.**

    -- BAD
    markDirty model =
      let
        model_ =
          { model | dirty = True }
      in
        model_

    -- GOOD
    markDirty model =
      let
        dirtyModel =
          { model | dirty = True }
      in
        dirtyModel

**Use anonymous function `\_ ->` over `always`**

It's more concise, more recognizable as a function, and makes it easier to change your mind later and name the argument.

    -- BAD
    on "click" Json.value (always Clicked)

    -- GOOD
    on "click" Json.value (\_ -> Clicked)

**Only use backward function application `<|` when parentheses would be awkward**

    -- BAD
    foo <| bar <| baz qux

    -- GOOD
    foo (bar (baz qux))

and

    -- BAD
    customDecoder string
      (\str ->
        case str of
          "one" ->
            Result.Ok 1

          "two" ->
            Result.Ok 2

          "three" ->
            Result.Ok 3
        )

    -- GOOD
    customDecoder string <|
      \str ->
        case str of
          "one" ->
            Result.Ok 1

          "two" ->
            Result.Ok 2

          "three" ->
            Result.Ok 3

**Use `case..of` over `if` where possible**

`case..of` is clever as it will generate more efficient JS, and it also allows you to catch unmatched patterns at compile time. It's also cheap to extend this data with something more useful later on, like if you need to add another branch. This saves code diffs.

**If a function can be pulled outside of a let binding, then do it**

Giant let bindings hurt readability and performance. The less nested a function, the less functions are used in generated code.

The update function is especially prone to get longer and longer: keep it as small as possible. Smaller functions that don't live in a let binding are more reusable.

**If your application has too many constructors for your `Msg` type, consider refactoring**

Large `case..of` statements hurts compile time. It might be possible that some of your constructors can be combined, for example `type Msg = Open | Close` could actually be `type Msg = SetOpenState Bool`.

#### Ports

The communication with **JS** is done via ports. There should be only one file which describes ports. The file is located in `app/elm/Shared/Js/Ports.elm`. **Elm** communicates with **JS** by sending/receiving data. Data has following format.

*   `tag` - **String**, which is the message name for **Elm**/**JS** to identify the appropriate action.
*   `data` - **Json**, which describes the data passed via ports.

Module `Ports.elm` exposes two function `msgToJs` and `msgFromJs`. Which could be used to pass/receive data from **Elm**/**JS**.

Each module which wants to communicate with **JS** should defined it's functions/messages for the communication. E.g.

    import Ports as Ports
    import Effects.MsgFromJs as MsgFromJs
    import Extra.Platform.Cmd as Cmd

    moduleName : String
    moduleName =
        "My.Module.Name"

    type Msg
        = NoOp
        | MsgFromJs (List Msg)
        | LogError String

    update : Msg -> Model -> ( Model, Cmd Msg )
    update msg model =
        case msg of
            NoOp ->
                ( model, Cmd.none )

            MsgFromJs msgsFromJs ->
                ( model
                , Cmd.dispatchList msgsFromJs
                )

            LogError error ->
                ( model
                , Ports.reportToSentry error
                )

    type MsgForJs
      = ValueChanged String

    sendMsgToJs : MsgForJs -> Cmd msg
    sendMsgToJs msg =
        case msg of
            ValueChanged value ->
                Ports.msgToJs
                    { tag = "ValueChanged"
                    , data = Encode.string value
                    }

    subscriptions : Sub Msg
    subscriptions =
        Ports.msgFromJs (MsgFromJs.decodeJsData MsgFromJs LogError msgFromJsEffects)

    msgFromJsEffects : MsgFromJs.Effects Msg
    msgFromJsEffects =
        MsgFromJs.subscribe moduleName
            [ ( "SomeMsgFromJs"
              , MsgFromJs.succeed NoOp
              )
            ]

Where `sendMsgToJs` is a command which could be used in **update** phase.

#### Tests

Use `Fuzz` tests over unit tests.
