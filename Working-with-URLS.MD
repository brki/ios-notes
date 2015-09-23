Convert a dictionary of parameters into a URL-encoded query string:
```swift
func queryStringFromParams(params: [String: String], includingLeadingQueryIndicator: Bool = true) -> String? {
    if count(params) == 0 {
        return ""
    }
    var queryItems = map(params) { NSURLQueryItem(name:$0, value:$1) }
    let components = NSURLComponents()
    components.queryItems = queryItems
    var queryString = components.percentEncodedQuery
    if let qs = queryString {
        if includingLeadingQueryIndicator {
            return "?\(qs)"
        }
        return qs
    }
    return nil
}
```
