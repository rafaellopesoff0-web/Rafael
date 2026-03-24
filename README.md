/// @name Auto accept trades
/// @group Trading
/// @desc 
///   Automatically accepts trades.
///   The trade window won't be shown in-client,
///   and your inventory won't refresh until the script is canceled.
/// @author b7
/// @scripter 1.0.0-beta

OnIntercept((
  In.TradeOpen, In.TradeCompleted,
  In.TradeItems, In.TradeAccept,
  In.TradeConfirmation,
  In.InventoryAddOrUpdateFurni
), e => e.Block());

OnTradeCompleted(e => {
  ShowBubble(
    $"Received {e.PartnerOffer.CreditCount}c, "
    + $"{e.PartnerOffer.FurniCount} furni "
    + $" from {e.Partner.Name}"
  );
});

ShowBubble($"{EnsureInventory().Count()} items in inventory");

try {
  while (Run) {
    Status("Waiting for trade...");
    Receive(In.TradeOpen);
    while (Run) {
      Status("Waiting for partner to accept trade...");
      var packet = Receive((In.TradeAccept, In.TradeClose));
      if (packet.Header != In.TradeAccept) break;
      packet.ReadInt();
      bool accepted = packet.ReadInt() == 1;
      if (accepted) {
        Status("Accepting trade...");
        Delay(100);
        Send(Out.TradeAccept);
        Status("Waiting for trade confirmation...");
        packet = Receive((
          In.TradeConfirmation,
          In.TradeItems,
          In.TradeClose,
          In.TradeCompleted
        ));
        if (packet.Header == In.TradeItems) continue;
        if (packet.Header != In.TradeConfirmation) break;
        Status("Confirming trade...");
        Delay(100);
        Send(Out.TradeConfirmAccept);
        Status("Waiting for trade completion...");
        Receive(In.TradeClose);
        break;
      }
    }
  }
} finally { Send(In.InventoryInvalidate); }
