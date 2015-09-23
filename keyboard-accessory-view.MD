## Add keyboard accessory view for text field

This can be used to add a done button in a bar over the keyboard, for example for a numeric keypad:
```swift
// From http://stackoverflow.com/a/25140225/948341 :
func addDoneButton() {
    let keyboardToolbar = UIToolbar()
    keyboardToolbar.sizeToFit()
    let flexBarButton = UIBarButtonItem(barButtonSystemItem: .FlexibleSpace,
        target: nil, action: nil)
    let doneBarButton = UIBarButtonItem(barButtonSystemItem: .Done,
        target: view, action: Selector("endEditing:"))
    keyboardToolbar.items = [flexBarButton, doneBarButton]
    textField.inputAccessoryView = keyboardToolbar
}
```
