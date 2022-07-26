# Keepers

This struct interface defines collections of methods that change the state (`sdk.Context`) and produce the corresponding trace tables for zkp.


## SendKeeper

```go
type SendKeeper interface {
    ViewKeeper

    InputOutputCoins(ctx sdk.Context, inputs []types.Input, outputs []types.Output) error
    SendCoins(ctx sdk.Context, fromAddr sdk.AccAddress, toAddr sdk.AccAddress, amt sdk.Coins) error

    GetParams(ctx sdk.Context) types.Params
    SetParams(ctx sdk.Context, params types.Params) error

    IsSendEnabledDenom(ctx sdk.Context, denom string) bool
    SetSendEnabled(ctx sdk.Context, denom string, value bool)
    SetAllSendEnabled(ctx sdk.Context, sendEnableds []*types.SendEnabled)
    DeleteSendEnabled(ctx sdk.Context, denom string)
    IterateSendEnabledEntries(ctx sdk.Context, cb func(denom string, sendEnabled bool) (stop bool))
    GetAllSendEnabledEntries(ctx sdk.Context) []types.SendEnabled

    IsSendEnabledCoin(ctx sdk.Context, coin sdk.Coin) bool
    IsSendEnabledCoins(ctx sdk.Context, coins ...sdk.Coin) error

    BlockedAddr(addr sdk.AccAddress) bool
}
```

For example, `SendCoins` should be something like 

TODO: Edit this to produce trace tables.
```go
func (k BaseSendKeeper) SendCoins(ctx sdk.Context, fromAddr sdk.AccAddress, toAddr sdk.AccAddress, amt sdk.Coins) error {
	err := k.subUnlockedCoins(ctx, fromAddr, amt)
	if err != nil {
		return err
	}

	err = k.addCoins(ctx, toAddr, amt)
	if err != nil {
		return err
	}

	// Create account if recipient does not exist.
	//
	// NOTE: This should ultimately be removed in favor a more flexible approach
	// such as delegated fee messages.
	accExists := k.ak.HasAccount(ctx, toAddr)
	if !accExists {
		defer telemetry.IncrCounter(1, "new", "account")
		k.ak.SetAccount(ctx, k.ak.NewAccountWithAddress(ctx, toAddr))
	}

	// bech32 encoding is expensive! Only do it once for fromAddr
	fromAddrString := fromAddr.String()
	ctx.EventManager().EmitEvents(sdk.Events{
		sdk.NewEvent(
			types.EventTypeTransfer,
			sdk.NewAttribute(types.AttributeKeyRecipient, toAddr.String()),
			sdk.NewAttribute(types.AttributeKeySender, fromAddrString),
			sdk.NewAttribute(sdk.AttributeKeyAmount, amt.String()),
		),
		sdk.NewEvent(
			sdk.EventTypeMessage,
			sdk.NewAttribute(types.AttributeKeySender, fromAddrString),
		),
	})

	return nil
}
```

