# ADR 4: Multiple Genesis modes

## Changelog

- Sept 3, 2019: Initial draft

## Context

In the current SDK, when the `GenesisState` is initialized through a `InitGenesis` call, some of the `GenesisState` fields are loaded from a `genesis.json` file, whereas in other cases they are initialized (_i.e_ populated) if the corresponding object value from JSON is empty, undefined or its value is zero.

This may lead to confusion because as is currently programmed, `InitGenesis`
initializes _and_ loads the values at the same time. When in reallity they should be
two separated disctinct processes.

A recurring case of this problem is to check if a balance is empty and initialize it
if not (by iterating the state). This affects mostly the total [`Supply`](https://github.com/cosmos/cosmos-sdk/blob/13e5e18d77e6010a4566ce187f18207669345419/x/supply/genesis.go#L14-L24) and[`ModuleAccounts`](https://github.com/cosmos/cosmos-sdk/blob/13e5e18d77e6010a4566ce187f18207669345419/x/gov/genesis.go#L44-L50)
within each module.

## Decision

We propose to add a separate command to update a genesis file with the missing values/fields for each module `GenesisState`.

With this command, the user provides a `genesis.json` file with the minimum amount
of genesis fields required to start a chain. The rest is loaded directly from the
mandatory genesis parameters. In general, the genesis fields that are populated
are the ones checked by the simulation invariants.

If the given genesis file contains _any_ field that shouldn't be populated, the
execution should `panic` and exit. Similarly, the initialization of the app (_i.e_
`<app>d init`) should `panic` if any genesis field is undefined.

The flow of this process can be summarized in 3 steps:

  1. Import genesis file with missing fields
  2. Create the app and initialize genesis with flag to fill the missing values
  3. Export updated genesis file

To archieve this, every module needs to be able to generate the missing fields from
the existing state. For example, on `x/supply/genesis.go`,
[`InitGenesis`](https://github.com/cosmos/cosmos-sdk/blob/13e5e18d77e6010a4566ce187f18207669345419/x/supply/genesis.go#L12)
needs to be modified to the following:

```go
// add populateGenesis parameter
func InitGenesis(ctx sdk.Context, keeper Keeper, ak types.AccountKeeper,
data GenesisState, populateGenesis bool) {
  if populateGenesis {
    if !data.Supply.Total.Empty() {
      panic("total supply should be initialized with empty coins ('[]') on populate genesis mode")
    }

    ak.IterateAccounts(ctx,
      func(acc authtypes.Account) (stop bool) {
        data.Supply.Total = data.Supply.Total.Add(acc.GetCoins())
        return false
      },
    )
  }

  keeper.SetSupply(ctx, data.Supply)
}
```

As this is part of the `AppModule` pattern, we also need to update its `InitGenesis`:

```go
// ./x/<module_name>/module.go
//
// add populateGenesis parameter
func (am AppModule) InitGenesis(ctx sdk.Context, data json.RawMessage, populateGenesis bool) {
  InitGenesis(ctx, am.k, am.ak ,data, populateGenesis) // supply module InitGenesis
}
```

Upstream, the application module `Manager` calls each module's `InitGenesis` function:

```go
// ./types/module/module.go
//
// InitGenesis performs init genesis functionality for modules. If the populateGenesis boolean is true, it updates the missing values of each module's GenesisState from the mandatory fields given in the genesis file data.
func (m *Manager) InitGenesis(ctx sdk.Context, genesisData map[string]json.RawMessage,
populateGenesis bool) abci.ResponseInitChain {

  var validatorUpdates []abci.ValidatorUpdate
  for _, moduleName := range m.OrderInitGenesis {
    if genesisData[moduleName] == nil {
      continue
    }
    moduleValUpdates := m.Modules[moduleName].InitGenesis(ctx, genesisData[moduleName], populate)

    // validator updates
  }
  return abci.ResponseInitChain{
    Validators: validatorUpdates,
  }
}
```

We also need to add a flag `FlagPopulateGenesis = "populate-genesis"` on
`server/start.go` that is passed to the `AppCreator` (eg: `NewSimApp` or `NewGaiaApp`)
when the `start` command from the daemon is run. This tells the app to fill the
missing genesis properties with the mandatory fields:

```go
app := NewSimApp(logger, db, nil, true, 0, SetPopulateGenesis(populate))
```

Additionally, a new option function needs to be passed as parameter to `BaseApp` during its initialization:

```go
// ./baseapp/options.go
//
// SetPopulateGenesis sets an option to populate or not the missing values
// of the each module GenesisState from the mandatory fields given on the genesis file
func SetPopulateGenesis(pupulateGenesis bool) func(*BaseApp) {
  return func(bap *BaseApp) { bap.setPopulateGenesis(pupulateGenesis)}
}
```

Which requires the following additions to `BaseApp`:

```go
// ./baseapp/baseapp.go
type BaseApp struct {
  // ...
  populateGenesis bool // flag for pupulating the genesis state from the mandatory fields on genesis.json
  // ...
}

func (app *BaseApp) setPopulateGenesis(pupulateGenesis bool) {
  app.populateGenesis = pupulateGenesis
}
```

Finally, the `pupulate-genesis` command imports the provided genesis dile and passes
the parameter to the module `Manager` call to update the state and then export it:

```go
// TODO: params
func PopulateCmd(ctx *Context, appCreator server.AppCreator, appExporter server.AppExporter) *cobra.Command {
    cmd := &cobra.Command{
    Use:   "pupulate-genesis [path/to/genesis.json]",
    Short: "Fill the missing non-mandatory values from the genesis file",
    Long:  `Fill the missing non-mandatory values from the genesis file by
    iterating the state`,
    Args:  cobra.ExactArgs(1),
    RunE: func(_ *cobra.Command, args []string) error {
      // initialize args
      genFile = args[0]
      genFile := config.GenesisFile()

      if appCreator == nil {
        return errors.New("app creator constructor function cannot be undefined")
      }

      if appExporter == nil {
        return errors.New("app exporter function cannot be undefined")
      }

      // 1. Import genesis file with missing fields
      appState, err := codec.MarshalJSONIndent(cdc, mbm.DefaultGenesis())
      if err != nil {
        return errors.Wrap(err, "Failed to marshall default genesis state")
      }

      genDoc := &types.GenesisDoc{}
      if _, err := os.Stat(genFile); err != nil {
        if !os.IsNotExist(err) {
          return err
        }
      } else {
        genDoc, err = types.GenesisDocFromFile(genFile)
        if err != nil {
          return errors.Wrap(err, "Failed to read genesis doc from file")
        }
      }

      // 2. Create the app and initialize genesis with flag to fill missing values
      cfg := ctx.Config
      home := cfg.RootDir
      traceWriterFile := viper.GetString(flagTraceStore)

      db, err := openDB(home)
      if err != nil {
        return nil, err
      }

      traceWriter, err := openTraceWriter(traceWriterFile)
      if err != nil {
        return nil, err
      }

      // CONTRACT: baseapp must have the option to populate the genesis
      app := appCreator(ctx.Logger, db, traceWriter)


      // 3. Export updated genesis file
      appState, validators, err := appExporter(ctx.Logger, db, traceWriter, -1, false, []string{})
      if err != nil {
        return fmt.Errorf("error exporting state: %v", err)
      }

      doc, err := tmtypes.GenesisDocFromFile(ctx.Config.GenesisFile())
      if err != nil {
        return err
      }

      doc.AppState = appState
      doc.Validators = validators

      encoded, err := codec.MarshalJSONIndent(cdc, doc)
      if err != nil {
        return err
      }

      fmt.Println(string(sdk.MustSortJSON(encoded)))
      return nil
    }

    return cmd
}
```

## Status

Proposed

## Consequences

### Positive

- Reduce confusion when initializing the app, as now you can only provide a genesis file with all its fields defined.
- Add a command to populate a genesis file from the mandatory values.

### Negative

- The chain now only starts if all the fields of the genesis file are populated.
- Remove the current single option to partially fill in the genesis on the initialization (_i.e_ only with a few non-mandatory fields missing).
This can be replaced by the following flow:

  1. Use the new empty chain option to auto-generate the genesis fields
  2. Modify the fields desired before starting the chain

### Neutral

- Add an additional parameter to `BaseApp`.
- Apps must update their genesis files and `InitGenesis` implementations.

## References

- [#2862](https://github.com/cosmos/cosmos-sdk/issues/2862)
- [#4255 (comment)](https://github.com/cosmos/cosmos-sdk/pull/4255/files/6a00e9c6f4cdcddbfe599d9d7003d0d8d708b6c2#r292878792)
- [#4568](https://github.com/cosmos/cosmos-sdk/issues/4568)