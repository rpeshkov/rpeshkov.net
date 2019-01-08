---
title: "Golang Variable Shadowing"
date: 2017-06-28T23:18:08+02:00
categories: ["Go"]
tags: ["Go"]
---

Recently I've been playing with some code in Go. Code was quite simple, but what I wanted is to simplify error handling a little bit and make code more readable.

As many Go developers know, error handling in Go is usually done this way:

```go
func templateToFile(templateFilename string, filename string, data interface{}) error {

	f, err := os.OpenFile(filename, os.O_WRONLY|os.O_TRUNC|os.O_CREATE, 0666)
	if err != nil {
		return err
	}
	defer f.Close()

	t, err := template.ParseFiles(templateFilename)
	if err != nil {
		return err
	}

	return t.Execute(f, data)
}
```

I've used to it too, but I was wondering whether this can be simplified. So I decided to use named return values and reorganize my code to something like this

```go
func templateToFile(templateFilename string, filename string, data interface{}) (err error) {

	if f, err := os.OpenFile(filename, os.O_WRONLY|os.O_TRUNC|os.O_CREATE, 0666); err == nil {
		defer f.Close()

		if t, err := template.ParseFiles(templateFilename); err == nil {
			return t.Execute(f, data)
		}
	}
	return
}
```

As you can see, here I have named return variable `err` and in each operation we put something there so that if `os.OpenFile` fails, you'll get `err != nil` and it will be returned. Theoretically this looks nice, but it doesn't work because of variable shadowing. In this particular example `err` in return value, `err` in first if block and `err` in second if block are different variables!

My assumption that my code would work was based on this statement from [Golang Specification](https://golang.org/ref/spec):

> Redeclaration does not introduce a new variable; it just assigns a new value to the original.

That's true, redeclaration just assigns a new value to the original but there's a small note... if they're in the same block.

Information from the specification

> An identifier declared in a block may be redeclared in an inner block. While the identifier of the inner declaration is in scope, it denotes the entity declared by the inner declaration.

So, if you use short assignment (`:=`) to declare variable with the same name in inner block, you will have new variable that has the same name as in outer block and any changes of the value won't affect variable from outer block.

Really good sample of this behavior you can find in [50 Shades of Go](http://devs.cloudimmunity.com/gotchas-and-common-mistakes-in-go-golang/):

```go
func main() {
    x := 1
    fmt.Println(x)     //prints 1
    {
        fmt.Println(x) //prints 1
        x := 2
        fmt.Println(x) //prints 2
    }
    fmt.Println(x)     //prints 1 (bad if you need 2)
}
```

## Detect shadowing

Variable shadowing can be detected by `vet` command from Go. Call it this way:

```bash
go tool vet --shadow file.go
```

When you run this command, you'll see shadowing issues in the file you've provided.

Example:

```bash
main.go:14: declaration of "err" shadows declaration at main.go:13
```

## Conclusion

Shadowed variables is basic type of errors and easily can be detected. However, when you have never faced this error before, you can spend some time trying to understand what's going on in your code.

Be careful with shadowing and always vet your code!
