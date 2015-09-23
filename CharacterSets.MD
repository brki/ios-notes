## List all characters that are in a character set
```
/**
Get all of the characters that are in the given character set.

Based on http://stackoverflow.com/a/26611100/948341 .

Swift 2.0 compatible.
*/
func charsInCharset(charset: NSCharacterSet) -> [Character] {
    var array = [Character]()
    for (var plane: UInt8 = 0; plane <= 16; plane++) {
        if charset.hasMemberInPlane(plane) {
            for (var c: UTF32Char = UInt32(plane) << 16; c < (UInt32(plane)+1) << 16; c++) {
                if charset.longCharacterIsMember(c) {
                    array.append(Character(CChar32(c)))
                }
            }
        }
    }
    return array
}
```

Usage:
```
charsInCharset(NSCharacterSet.URLPathAllowedCharacterSet())
```
