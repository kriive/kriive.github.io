---
title: "filepath.Clean: terms and conditions apply"
date: 2023-09-01T15:00:00+02:00
---
At least once in their career, every Go programmer had to write code that 
interacted with the filesystem. For example, if our application needed
to handle some documents, we probably had to write code that handled the 
files and saved it under a directory.

We may have ended up writing code that resemble this:
```go
p := filepath.Join(uploadsDirectory, userChosenFileName)
f, err := os.OpenFile(p, os.O_RDWR|os.O_CREATE, 0755)
if err != nil {
  return err
}
f.Write(userPayload)
// ...
```
For the expert eye, this pattern looks suspicious and error prone: what if
a malicious user chose to name their file `../../../../home/kriive/.ssh/authorized_key`? It would end up overwriting my `authorized_keys` file
with an attacker controlled value and letting them SSH into my box.
This attack is commonly referred to as *path traversal attack*.

This is a fairly common scenario and most developers end up using safeguards
against this. In Go they may resort to using one of the other `filepath` 
module functions, such as `filepath.Clean` and `filepath.Rel`.
In this article I will focus on the first function.

One can think that slapping `filepath.Clean` on the user input may be 
sufficient, like this example.
```go
// This is insecure!
p := filepath.Join(uploadsDirectory, filepath.Clean(userChosenFileName))
f, err := os.OpenFile(p, os.O_RDWR|os.O_CREATE, 0755)
if err != nil {
  return err
}
f.Write(userPayload)
// ...
```
This is not secure. If we read [the documentation of `filepath.Clean`](https://pkg.go.dev/path/filepath#Clean), it is stated that the function will:
> [...]
> Eliminate .. elements that begin **a rooted path**: that is, replace "/.." by "/" at the beginning of a path, assuming Separator is '/'.

So, this function strips the extra .. only if our path begins with a slash!
If the attacker choses `../../../../home/kriive/authorized_keys`, it leaves
the path as it is and enables the *path traversal attack*!

## How to fix?
If you want to use `filepath.Clean` safely, you must ensure that the user
supplied path begins with a slash. Even if there are two or more slashes, 
`filepath.Clean` will strip them out.

```go
// This is ok
p := filepath.Join(uploadsDirectory, filepath.Clean("/"+userChosenFileName))
f, err := os.OpenFile(p, os.O_RDWR|os.O_CREATE, 0755)
if err != nil {
  return err
}
f.Write(userPayload)
// ...
```

[Here](https://go.dev/play/p/X2sY6ykdltL) is an interactive example that shows the vulnerable pattern.
