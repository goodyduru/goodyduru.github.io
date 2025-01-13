+++
date = '2025-01-13T16:42:13+01:00'
draft = false
title = 'Using Must, Funcs, and ParseFiles with multiple files in Go templates'
+++
I wanted to add a few custom template functions in my first Go web application, which I'm currently building. Something like this
```go
// Setup template
funcMap := template.FuncMap{
    "trim": strings.Trim,
}
customTemplate := template.Must(template.New("test").Funcs(funcMap).ParseFiles("views/base.html", "views/submit.html"))
.
.
// Call template
d := ....
customTemplate.Execute(w, d)
```
<!--more-->
I found out that this was the only way to generate a page with custom functions in a performant way. After code compilation and execution, the browser screen was empty.

It turns out that the `customTemplate` object contains more than one inner template object, including both the ones created by [New](https://pkg.go.dev/html/template#New) and [ParseFiles](https://pkg.go.dev/html/template#ParseFiles). Calling `Execute` applies the template object created by `New`. Unfortunately, this wouldn't include the HTML string returned by `ParseFiles`. This was why my browser screen was empty. To fix it, I used the `ExecuteTemplate` function like this:
```go
customTemplate.ExecuteTemplate(w, "base.html", d)
```
The [`ExecuteTemplate` function](https://pkg.go.dev/html/template#Template.ExecuteTemplate) allows me to pick a specific template object, which, in this case, will be one returned by `ParseFiles`.

I could also set the parameter in the `New` function to what I expect `ParseFiles` template object name to be (in this case, `base.html`). This will allow me to just call the `Execute` function. Here's how it will be
```go
customTemplate := template.Must(template.New("base.html").Funcs(funcMap).ParseFiles("views/base.html", "views/submit.html"))
....
customTemplate.Execute(w, d)
```
Both the `New` and `ParseFiles` template objects have the same name, `ParseFiles` overwrites the template object allocated by `New` with its own. `Execute` now applies the `ParseFiles` template object which contains the HTML string I want.

This [Stackoverflow answer](https://stackoverflow.com/questions/41176355/go-template-name/41187671#41187671) has way more details on Go Template names.