This describes how, using Swift in iOS 8.x, to use a UIScrollView and auto-layout constraints to present a screen
with editable text fields.  The view fits in the window; e.g. the content is not so long as to need scrolling.

When the keyboard pops up or changes size, the currently active text field will be scrolled into view if necessary
(e.g. if it would otherwise be hidden beneath the keyboard).

## In the storyboard
In the storyboard, add a UIScrollView as a child of the main view.  Add a child view to the UIScrollView, which we'll
refer to as the content view.

* Pin the UIScrollView to all 4 main view edges (Trailing, Leading, Bottom Layout Guide, Top Layout Guide)
* Pin the content view to all 4 UIScrollView edges
* Give the content view a height and width for use in the storyboard (e.g. placeholder constraints; check "Remove at build time")

Inside the content view, layout the text fields and other elements.

## Storyboard outlets
Create an outlet in the view controller for:

* the UIScrollView (the code below calls it ``scrollView``)
* the content view (the code below calls it ``contentView``)

## In the ViewController

The important parts are:
```swift

// The outlets and some variables:
@IBOutlet weak var scrollView: UIScrollView!
@IBOutlet weak var contentView: UIView!

var isRotating = false  // Will be true during rotation
var heightConstraint: NSLayoutConstraint?  // Height constraint, will be adjusted when rotating
var activeTextField: UITextField?  // Keeps track of the active text field.

override func viewDidLoad() {
    super.viewDidLoad()

    // Set textField delegates for *all* text fields used in the view.
    textSearchField.delegate = self
    latitudeField.delegate = self
    longitudeField.delegate = self

    // Add constraints for the content view, which can not be done in the storyboard (pinning left/right edges to self.view,
    // setting the height constraint based on the view size and the status bar size).
    let leftConstraint = NSLayoutConstraint(
        item: contentView,
        attribute: NSLayoutAttribute.Leading,
        relatedBy: NSLayoutRelation.Equal,
        toItem: view,
        attribute: NSLayoutAttribute.Leading,
        multiplier: 1,
        constant: 0)
    let rightConstraint = NSLayoutConstraint(
        item: contentView,
        attribute: NSLayoutAttribute.Trailing,
        relatedBy: NSLayoutRelation.Equal,
        toItem: view,
        attribute: NSLayoutAttribute.Trailing,
        multiplier: 1,
        constant: 0)

    // Set the height constraint for the content view, so that the auto-layout constraints
    // for the content view subviews will work correctly.
    // This may need some adjustment if a navigation bar / tab bar / tool bar is present.
    heightConstraint = NSLayoutConstraint(
        item: contentView,
        attribute: NSLayoutAttribute.Height,
        relatedBy: NSLayoutRelation.Equal,
        toItem: nil,
        attribute: NSLayoutAttribute.NotAnAttribute,
        multiplier: 1,
        constant: UIScreen.mainScreen().bounds.size.height - UIApplication.sharedApplication().statusBarFrame.size.height)

    view.addConstraints([leftConstraint, rightConstraint, heightConstraint!])
}

override func viewWillAppear(animated: Bool) {
    super.viewWillAppear(animated)
    NSNotificationCenter.defaultCenter().addObserver(self, selector: "keyboardChangingSize:", name: UIKeyboardWillChangeFrameNotification, object: nil)
    NSNotificationCenter.defaultCenter().addObserver(self, selector: "statusBarChangingSize:", name: UIApplicationWillChangeStatusBarFrameNotification, object: nil)
}

override func viewWillDisappear(animated: Bool) {
    super.viewWillDisappear(animated)
    NSNotificationCenter.defaultCenter().removeObserver(self)
}

/**
The status bar may or may not be visible in the new orientation.
Keep track of the rotation status.  When the keyboard is present, some UIKeyboardWillChangeFrameNotification
notifications are sent during rotation, but we can ignore those.
*/
override func viewWillTransitionToSize(size: CGSize, withTransitionCoordinator coordinator: UIViewControllerTransitionCoordinator) {
    super.viewWillTransitionToSize(size, withTransitionCoordinator: coordinator)
    isRotating = true
    coordinator.animateAlongsideTransition(
        { context in
            self.setHeightConstraintIfNeeded()
        },
        completion: { context in
            self.isRotating = false
    })
}

/**
Set the content view height constraint based on the space available.
*/
func setHeightConstraintIfNeeded() {
    if let constraint = heightConstraint {
        let currentStatusBarHeight = UIApplication.sharedApplication().statusBarFrame.size.height
        if currentStatusBarHeight != constraint.constant {
            constraint.constant = UIScreen.mainScreen().bounds.size.height - currentStatusBarHeight
        }
    }
}

/**
Keep track of the currently active text field.
*/
func textFieldDidBeginEditing(textField: UITextField) {
    activeTextField = textField
}

/**
Unset the currently active text field when the text field resigns as a first responder. 
*/
func textFieldDidEndEditing(textField: UITextField) {
    if activeTextField == textField {
        activeTextField = nil
    }
}

/**
Reset the content view's height when the status bar changes size.
*/
func statusBarChangingSize(notification: NSNotification) {
    setHeightConstraintIfNeeded()
}

/**
Change the UIScrollView's contentInset when the keyboard appears / disappears / changes size.
If necessary, scroll so that the currently active text field is visible.
*/
func keyboardChangingSize(notification: NSNotification) {
    if isRotating {
        // No need to handle notifications during rotation
        return
    }
    if let userInfo = notification.userInfo as [NSObject: AnyObject]? {
        if let endFrame = (userInfo[UIKeyboardFrameEndUserInfoKey] as? NSValue)?.CGRectValue() {
            let convertedEndFrame = view.convertRect(endFrame, fromView: view.window)
            if convertedEndFrame.origin.y == view.bounds.height {
                // Keyboard is hidden.
                let contentInset = UIEdgeInsetsZero
                scrollView.contentInset = contentInset
                scrollView.scrollIndicatorInsets = contentInset
            } else {
                // Keyboard is visible.
                let animationDuration = userInfo[UIKeyboardAnimationDurationUserInfoKey] as? Double ?? 0.0
                let animationOption = userInfo[UIKeyboardAnimationCurveUserInfoKey] as? UIViewAnimationOptions ?? UIViewAnimationOptions.TransitionNone
                let keyboardTop = convertedEndFrame.origin.y

                if let textField = activeTextField {
                    var textFieldRect = textField.convertRect(textField.bounds, toView: view)
                    let textFieldBottom = textFieldRect.origin.y + textFieldRect.height
                    let offset = textFieldBottom - keyboardTop
                    if offset > 0 {
                        let contentInset = UIEdgeInsets(top:0.0, left:0.0, bottom:convertedEndFrame.height, right:0.0)
                        scrollView.contentInset = contentInset
                        scrollView.scrollIndicatorInsets = contentInset
                        UIView.animateWithDuration(
                            animationDuration,
                            delay: 0.0,
                            options: animationOption,
                            animations: {
                                self.scrollView.scrollRectToVisible(textFieldRect, animated: false)
                            },
                            completion: nil)
                    }
                }
            }
        }
    }
}
```

## References

* http://spin.atomicobject.com/2014/03/05/uiscrollview-autolayout-ios/
* https://developer.apple.com/library/ios/documentation/StringsTextFonts/Conceptual/TextAndWebiPhoneOS/KeyboardManagement/KeyboardManagement.html
