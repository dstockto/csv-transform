CSV-Transform
===

This project is intended to make CSV->CSV transformations easy to define and execute. The program uses a "recipe" which is a program that is designed and intended to be simple and easy to understand, even if you're not a developer.

There are two main modes of operation: transform and generate.

Transform
--
This is intended to be the main mode. 

`csv-transform transform /path/to/input.csv /path/to/output.csv /path/to/recipefile`

The program requires an input CSV file, an output CSV file and a recipe file. CSVs can be whatever you want as long as they are legitimate, parseable CSVs.

Please see the recipes section for information about how to build recipes for the program.

Generate
--
The generate function can use recipe file to generate sample data files using data generation functions.

`csv-transform generate /pat/to/output.csv /path/to/recipe --lines=<number of lines, default 100> [--with-header]`

The recipes are built in the same way as normal transformations, other than it probably makes no sense to include any recipes that reference columns since there's no input file.
Here's an example generate recipe with header info:

```
$first <- fake("first name")
$last <- fake("last name")
!1 <- "first name"
1 <- $first
!2 <- "last name"
2 <- $last
!3 <- "fullname"
3 <- $first + " " $last -> uppercase
!4 <- "address"
4 <- fake("address")
!5 <- "city"
5 <- fake("city")
!6 <- "state"
6 <- fake("state")
!7 <- "email"
7 <- fake("email")
```

As you can see, this could be a useful tool for generating data files with test data for importing into various systems that allow importation of CSV files. The first two lines includes creation of variables that can then be used later. The `$first` and `$last` variables are used directly in their own columns, but also combined and piped through the uppercase function in order to generate a full uppercase name that is consistent with the other two fields.

Recipes
==

There are three types of value can have data assigned: headers, columns and variables.
Headers are processed only for the first line of output and only if they are turned on with the `--withHeaders` option.

Variables allow you to store information that can be reused within a column, and can be used to allow for more complex operations than would otherwise be allowed with what you can do with a single column's recipe.

Within a recipe file, it's easy to identify columns, headers, variables, and functions because they are very limited in how they can be defined.

Columns consist of only digits. If you see a number by itself, it's a column reference.

Headers are an exclamation point followed by a column number with no spaces, like `!2`. If you want to add a column header for an inserted column, these can be useful. You could also use them to change existing headers. You can use all the features of a recipe when defining a header, but remember, for transformations, it will running against existing header values. For generate recipes, there are no incoming columns, so it doesn't make any sense to try to use column references.

Variables can be identified because they start with a `$` and consist of letters, for example `$firstname`.

Functions consist of only letters. They can either be just letters, or they can potentially require arguments which should be provided inside parentheses. If there are more than one, they should be separated by commas. Arguments to a function can be columns, variables or literals.

Literals are plain text. They are wrapped in double quotes. For example, "Header 3". You can, if needed, include quotes inside a literal by escaping them with a backslash, like so `"this \" <- is a quote"`. If you want to include a backslash, you must also escape it as well, which means type two backslashes - `"backslash \\"`.

If you want to put comments in your recipe, you can! Use `#` to indicate a comment. This can be either at the start of a line, which indicates the entire line is a comment, or at the end of a recipe line. If you put a '#' in literal, it will not be considered a comment character. However, if you try to add a comment before a recipe line is potentially complete, you will see an error when trying to run the recipe.

The next important item is the assignment operator. It is a left arrow created with a less-than and a dash - `<-`. Each recipe line will start with either a variable, column or header followed by the assignement operator, followed by your recipe to build the value that should go in the output.

The rest of the line consists of combinations of columns, variables, literals and functions. They can be combined using a right arrow `->` which indicates whatever has been built should be passed along into the next thing (typically this is going to be a function) or you can use `+` to combine the values on the left and the right of the plus. Recipes are executed strictly left-to-right. This means if you have more complex operations that would require out of order processing, you should consider variables in order to simplify the operation into something that can be executed linearly.

Let's take a look at some simple recipes. We'll be looking at single lines at a time, but a full recipe file will likely consist of many of these:

`# full line comment`

This line will not do anything for the output, but it can be helpful to leave notes for yourself or others who might be helping with maintaining the recipe.

`1 <- 1`

This is one of the simplest recipes. It means that in the output CSV, the first column will come from the first column of the input CSV.

`1 <- 2`

In this recipe, we're moving a column over. Column 2 of the input file will be placed into column 1 of the output file.

`$first <- 3`

In this recipe, we're taking column 3 of each row and placing it into a variable that we can reference later for other recipe lines. This can help with ensuring consistency or efficiency. Variables can also help to simplify more complex expressions into expressions that can execute linearly, left-to-right.

`5 <- $first`

This recipe has the output column 5 receiving the value of the `$first` variable. 

`!4 <- "fruit picked"` 

This recipe sets a header for column 4 to the phrase `fruit picked`. The exclamation point in front of 4 indicates this is a header definition, and the quotes around "fruits picked" mean we're using this value as provided.

While there is quite a lot that can be done with just the examples about, we can get even more power when utilizing pipe (->) and join (+).

`1 <- 2 + 3 # concatenate 2 and 3 into 1`

This recipe reads the value of column 2, tacks on column 3 to the end of it and puts the result in column 1. I've also included a comment at the end of the recipe to show how that can work as well.

`1 <- 1 -> uppercase`

This recipe leaves column 1 in the first position, but transforms the value through the uppercase function which transforms any lowercase characters into uppercase.

`3 <- 3 + uppercase`

This one may be a bit tricky, but it's different than the example above. Instead of using a pipe operator, it's using join. This means instead of just uppercasing the column, it combines the original value with the uppercased value. For example if the original column value was "foo", then this recipe would result in "fooFOO". Using a pipe would result in only "FOO".

`2 <- 2 -> lowercase + " particles"`

This recipe will lowercase column 2 and then add the word "particles" with a space. This result ends up in column 2 for the output.

The next few recipes, while written differently, will all result in the same outcome.

```
1 <- 1 -> uppercase
1 <- uppercase(1)
1 <- uppercase(?)
```

This example introduced one new concept I haven't mentioned before -- the placeholder, represented by ?. This is implicitly what's used from one step in a recipe to the next and passed into functions for which you do not provide arguments.

This final example is a bit silly but may help illustrate the placeholder a bit.

`
1 <- 1 + ? + ? + uppercase(?)
`

Suppose column 1 contained "apple". This means after that lookup, the placeholder would contain "apple". Then we combine the placeholder value with the next operation which is the placeholder value. That means after `1 + ?` the placeholder would contain "appleapple". This is the new placeholder value. Then the next `+ ?` happens which takes "appleapple" and comboines it with the same, resulting in "appleappleappleapple". After the final operation, which takes the 4xapple and combines with an uppercase version of the same, the placeholder and the ultimate result will be "appleappleappleappleAPPLEAPPLEAPPLEAPPLE" or, "apple" repeated 8 times, with the first 4 lowercase and the last 4 uppercase. I don't know why you'd ever want or need to do this, but... you could I guess.

There's more you can do, but it would be impossible to provide examples for all of them. Please see the functions section for what the provided functions do to learn more about the possibilities.

Available Functions
==

Functions can be hard to understand at first but the important thing to know is that they will all return a "string" or character value. They may require some sort of input or possibly even configuration and I'll try to document that here. If the function can accept and return a string, it will also accept a column or variable or literal value. If it doesn't care about what you provide, I'll indicate that with the `?` or placeholder value, but you can provide one of the other value options. If it does matter what you provide, I'll document that as well, and will probably name the parameter something to indicate its value. You still could provide a variable or column or literal value, but it would need to match whatever the function is expecting. 

Finally, if the function takes a final parameter of `?` then you can leave it off of your recipe and the program will automatically provide it. In fact, it will automatically provide the placeholder value for all arguments that you don't explicitly provide. This may not be what you want though and the output may not be correct or a function that expects certain input may result in an error. If a function does not need any parameters and won't use them if you provide them, I'll indicate that with empty parens. You can leave those off too.


* uppercase(?) - transforms characters in the value to uppercase - ex uppercase("apple") is APPLE.
* lowercase(?) - transforms characters in the value to lowercase - ex lowercase("LOWER") is lower.
* today() - returns today's date in YYYY-mm-dd format, ex 2021-08-23
* add(?, ?) - accepts two values that should be numerical and returns a string representing the sum of those two values. Providing non-numerical values will probably not do what you want. Remember, `add(2, 3)` is not 5, it's the sum of the values in columns 2 and 3.
* subtract(?, ?) - returns the value of the first parameter minus the second. All the caveats that apply to add apply here.
* multiply(?, ?) - returns the product of the two provided numerical values
* divide(?, ?) - provides the result of first value divided by the second. They should of course be numbers and the second value should not be zero unless you want to cause damage to the space-time continuum.
* normalize_date(format, date) - This function can accept a date in the provided `format` and return a string of that date in a format that other functions that need dates can utilize.
* format_date(format, date) - Use this at the end of a line of date operations to get a date in a format that you want. 
* if_after(after, not_after, date) - This function will return the `after` value if today is after the provided `date`, or the `not_after` value if today is before `date`.
* if_same(same, not_same, x, y) - If `x` and `y` are the same, this function returns the `same` value. If they are not, it returns the `not_same` value.
* only_digits(?) - returns all digit characters from the provided value
* not_digits(?) - strips all digit characters from the provided value
* trim(?) - removes whitespace from the provided value
* first_chars(num, ?) - returns the first `num` characters of a string
* last_chars(num, ?) - returns the last `num` characters of a string