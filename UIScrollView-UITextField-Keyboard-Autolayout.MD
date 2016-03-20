This describes how, using Swift 2.1 in iOS 9.x, to use a UIScrollView and auto-layout constraints to present a screen
with editable text fields.  The view fits in the window; e.g. the content is not so long as to need scrolling.

When the keyboard pops up or changes size, the currently active text field will be scrolled into view if necessary
(e.g. if it would otherwise be hidden beneath the keyboard).

## In the storyboard
In the storyboard, add a UIScrollView as a child of the main view.  Add a child view to the UIScrollView, which we'll
refer to as the content view.  You probably want to uncheck the 'Bounces' option, so that the view isn't draggable
when it all fits on one screen.

* Pin the UIScrollView to all 4 main view edges (Trailing, Leading, Bottom Layout Guide, Top Layout Guide)
* Pin the content view to the top and bottom UIScrollView edges
* Give the content view a width (for example width = view.width)
* Give the content view a fixed height constraint (this value will be adjusted at runtime)

Inside the content view, layout the text fields and other elements.

## Storyboard outlets
Create an outlet in the view controller for:

* the UIScrollView (the code below calls it ``scrollView``)
* the content view height constraint (the code below calls it ``contentViewHeightConstraint``)

## In the ViewController

The important parts are:
```swift

class YourViewController: UIViewController {

	// The outlets and some variables:
	@IBOutlet weak var scrollView: UIScrollView!
	@IBOutlet weak var contentViewHeightConstraint: NSLayoutConstraint!

	var isRotating = false             // Will be true during rotation
	var activeTextField: UITextField?  // Keeps track of the active text field.

	override func viewDidLoad() {
	    super.viewDidLoad()

	    // Set textField delegates for *all* text fields used in the view.
	    textSearchField.delegate = self
	    latitudeField.delegate = self
	    longitudeField.delegate = self
	}

	override func viewWillAppear(animated: Bool) {
	    super.viewWillAppear(animated)

		// If there's a bottom toolbar, let it size itself appropriately for the current orientation:
		// bottomToolbar.invalidateIntrinsicContentSize()

		// Register for the keyboard and status bar size-changing notifications:
	    registerForNotifications()
	}

	override func viewDidAppear(animated: Bool) {
		super.viewDidAppear(animated)

		// Set the height constraint appropriately for the current layout:
		setHeightConstraintIfNeeded()
	}

	override func viewWillDisappear(animated: Bool) {
		super.viewWillDisappear(animated)
		unregisterForNotifications()
	}
}


// MARK: keeping the text field visible while editing

extension YourViewController {

	/**
	The status bar may or may not be visible in the new orientation.
	Keep track of the rotation status.  When the keyboard is present, some UIKeyboardWillChangeFrameNotification
	notifications are sent during rotation, but we can ignore those.
	*/
	override func viewWillTransitionToSize(size: CGSize, withTransitionCoordinator coordinator: UIViewControllerTransitionCoordinator) {
		super.viewWillTransitionToSize(size, withTransitionCoordinator: coordinator)

		// Before the animation is a good time to call invalidateIntrinsicContentSize() on a toolbar.
		// This makes it's new height available during the animation.
		// bottomToolbar.invalidateIntrinsicContentSize()

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

	func registerForNotifications() {
		NSNotificationCenter.defaultCenter().addObserver(self, selector: "keyboardChangingSize:", name: UIKeyboardWillChangeFrameNotification, object: nil)
		NSNotificationCenter.defaultCenter().addObserver(self, selector: "statusBarChangingSize:", name: UIApplicationWillChangeStatusBarFrameNotification, object: nil)

	}

	func unregisterForNotifications() {
		NSNotificationCenter.defaultCenter().removeObserver(self)
	}

	/**
	The content view height.
	*/
	var contentViewHeight: CGFloat {
		let windowHeight = UIScreen.mainScreen().bounds.size.height
		return windowHeight - topBarHeight - bottomBarHeight
	}

	// This may need some adjustment if a navigation bar / tab bar / tool bar is present.
	var topBarHeight: CGFloat {
		return UIApplication.sharedApplication().statusBarFrame.size.height
	}

	// This may need some adjustment if a tab bar / tool bar is present.
	// Note that a toolbar may have a different height in landscape / portrait mode.
	var bottomBarHeight: CGFloat {
		return CGFloat(0)
		// return bottomToolbar.frame.height
	}

	/**
	Set the content view height constraint based on the space available.
	*/
	func setHeightConstraintIfNeeded() {
		if let constraint = contentViewHeightConstraint {
			let currentHeight = contentViewHeight
			if currentHeight != constraint.constant {
				constraint.constant = currentHeight
			}
		}
	}

	/**
	Reset the content view's height when the status bar changes size.
	*/
	func statusBarChangingSize(notification: NSNotification) {
		if !isRotating {
			setHeightConstraintIfNeeded()
		}
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

		guard let userInfo = notification.userInfo as [NSObject: AnyObject]?,
			endFrame = (userInfo[UIKeyboardFrameEndUserInfoKey] as? NSValue)?.CGRectValue() else {
				// Needed information not available.
				return
		}

		let convertedEndFrame = view.convertRect(endFrame, fromView: view.window)
		if convertedEndFrame.origin.y == view.bounds.height {

			// Keyboard is hidden.
			let contentInset = UIEdgeInsetsZero
			scrollView.contentInset = contentInset
			scrollView.scrollIndicatorInsets = contentInset

		} else {
			// Keyboard is visible.
			guard let textField = activeTextField else {
				// No active text field ...
				return
			}

			let keyboardTop = convertedEndFrame.origin.y
			let textFieldRect = textField.convertRect(textField.bounds, toView: view)
			let textFieldBottom = textFieldRect.origin.y + textFieldRect.height
			let offset = textFieldBottom - keyboardTop
			guard offset > 0 else {
				// Text field is already above the top of the keyboard.
				return
			}

			// Adjust the scroll view content inset.
			let contentInset = UIEdgeInsets(top:0.0, left:0.0, bottom:convertedEndFrame.height, right:0.0)
			scrollView.contentInset = contentInset
			scrollView.scrollIndicatorInsets = contentInset

			// Animate the text field into view.
			let animationDuration = userInfo[UIKeyboardAnimationDurationUserInfoKey] as? Double ?? 0.0
			let animationOption = userInfo[UIKeyboardAnimationCurveUserInfoKey] as? UIViewAnimationOptions ?? UIViewAnimationOptions.TransitionNone
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

```

## Originally inspired by:

* http://spin.atomicobject.com/2014/03/05/uiscrollview-autolayout-ios/
* https://developer.apple.com/library/ios/documentation/StringsTextFonts/Conceptual/TextAndWebiPhoneOS/KeyboardManagement/KeyboardManagement.html
