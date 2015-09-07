This can be used to keep secret keys etc. out of git.

The Config.config dictionary defines key / value pairs.
In a non-git-tracked file, create a Config extension that
implements OverriddenConfig.

app/config.swift:
```
protocol OverriddenConfig {
    func overrideConfig()
}

class Config {
    static let sharedInstance = Config()
    var config: [String: AnyObject] = [:]
    init() {
        if let overrideable = self as? OverriddenConfig {
            overrideable.overrideConfig()
        }
    }
}
```

app/config-local.swift:
```
extension Config: OverriddenConfig {
    func overrideConfig() {
        config["foo"] = "bar"
    }
}
```

Using it:
```
let config = Config.sharedInstance.config
```

