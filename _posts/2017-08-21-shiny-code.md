---
title: Code organization with Shiny
date: 2017-08-21T23:00:00Z
layout: post
path: "/shiny-code/"
tags:
  - R
  - shiny
---

### Use Case

Firstly, what is Shiny? Essentially, Shiny is an R add-on package that includes a web server and a UI engine, both of which are programmable inside R.

Would I use Shiny if I were trying to deploy a fully-featured website for mass public consumption? Probaly not. But then again, I wouldn't necessarily try to platformize anything written in R<sup>[1](#fn1)</sup>.

On the other hand, Shiny can do a lot of stuff, and can be a really powerful tool for the right use-case. I usually use Shiny as a very powerful `manipulate()` - a great interactive data browser and visualizer for you (and your boss).

### Basic App

While using Shiny, typically, you'll start off with a UI description `ui.R` and a server routine `server.R`. RStudio has a default app they use for visualizing Old Faithful data using a histogram with a variable number of bins. Here's their `server.R`:

```R
library(shiny)

# Define server logic required to draw a histogram
shinyServer(function(input, output) {
   
  output$distPlot <- renderPlot({
    
    # generate bins based on input$bins from ui.R
    x    <- faithful[, 2] 
    bins <- seq(min(x), max(x), length.out = input$bins + 1)
    
    # draw the histogram with the specified number of bins
    hist(x, breaks = bins, col = 'darkgray', border = 'white')
    
  })
  
})
```

And the corresponding `ui.R`:

```R
library(shiny)

# Define UI for application that draws a histogram
shinyUI(fluidPage(
  
  # Application title
  titlePanel("Old Faithful Geyser Data"),
  
  # Sidebar with a slider input for number of bins 
  sidebarLayout(
    sidebarPanel(
       sliderInput("bins",
                   "Number of bins:",
                   min = 1,
                   max = 50,
                   value = 30)
    ),
    
    # Show a plot of the generated distribution
    mainPanel(
       plotOutput("distPlot")
    )
  )
))
```

And Poof! it works<sup>[2](#fn2)</sup>. 

### Basic Error Hazards

Problem number one: The server and the UI don't directly talk to each other. See in `ui.R`, in the `sliderInput()` call? The first argument is `"bins"`, that's what sets the name of the attribute in the `input` list in `server.R`. Now how are you supposed to know that the `input` list is supposed to have `input$bins`? And the same goes for knowing the name of the plot for `plotOutput()` is `"distplot"`.

When Shiny first came out, there was really no way - you constantly had to be switching back and forth between the server and the ui files to make sure you were getting the names right. Nowadays, they've improved tab completion so it can help you out, but it's still difficult, especially as your number of ui inputs and outputs increases. There is still pretty huge danger for typos, and Shiny's debugging doesn't help much. For example, take that `input$bins`, and rename it to `input$bits`. You get this:

```
Warning: Error in seq.default: argument 'length.out' must be of length 1
Stack trace (innermost first):
    101: seq.default
    100: seq
     99: renderPlot [path/server.R#19]
     89: <reactive:plotObj>
     78: plotObj
     77: origRenderFunc
     76: output$distPlot
      1: runApp
```

At the very least, in this example, it points you to the correct line of where the error is found, but it's not very clear what it actually is. In other cases, it's even less clear. That's why (see all of my other blog posts) I generally like to have some kind of enforecment of the names and types of list attributes. I just don't trust myself to get them correct.

### Shiny Code Organizational Hazards

Here's one worse though - What do you do when your codebase gets big? Originally, they actually suggested using `source`. That's pretty much the worst, most hack-y thing I could think of. How is your `ui` or `server` file supposed to know what's actually there in the other file? It's not! Not a good way to program. Sure, it works when it works. But I like to program really defensively. I try to ensure stuff works from the get-go as best as I can (obviously). I like to put in defenses like ensuring my attribute names are right and throwing errors when they aren't. I like coding in a way such that when something doesn't work, the code will tell me why.

Fortunately, there's a bit of hope in this regard, [Shiny Modules](https://shiny.rstudio.com/articles/modules.html). Creating modular code this way is a huge step in the right direction, definitely out of the danger-zone. They still rely on passing strings as names of list attributes, which is what keeps me nervous.

### Testing and Expanding

Typically though, here's what I would do. The above `server` splits the Old Faithful data into bins, each of which can easily be described on a single line of code. That's ordinarily not going to be the case. Say for example, you wanted to filter those events, or have even a little bit of analysis? My recommendation - if you can turn something like that into a function, do it. Likely, you'll have more than one function if you do that, and that means write a package.

If I'm thinking of doing a data visualization with shiny, I'll have two RStudio windows open, one with my analysis, and the other as my Shiny app, as two separate `.Rproj`s, called something like "myanalysis_core" and "myanalysis_app".

In my opinion, Shiny Modules are great for organizing your UI code. You can only do that in the Shiny app. However I'd keep the vast majority of the `server.R` functionality packaged separately.

The reasons I'd do that are because it makes testing and code organization much easier. You're back using `testthat`, and your `server.R` file is crisp and clean.

Admittedly, there are ways of testing your app too. I like reserving those though for testing the UI functionality, not the data that are being manipulated. I'm much more comfortable testing packages than Shiny apps.

### A non-Shiny consideration

One last thought, and the other reason I keep anything that looks like analysis out of Shiny, it really is a choice based on what's the final product. It's happened a few times that I have performed an analysis, did data visualization in Shiny, and then needed to write up a report or presentation. There are great tools for doing those in R too using R Markdown. But if all of your analysis is *in* the Shiny app, you're going to need to need to copy and paste a lot of code. That's a bad idea of course, in case something changes.

Much better strategy, keep your data and analysis in a separate space from your App.



<a name="fn1">1:</a> You're thinking "Wait! I thought you loved R!" Yeah - I do. It's a great analysis tool, it has a lot of features and syntax that make data analysis a breeze. However, in my opinion, R is just not the right tool if you're trying to write something for mass consumption. It has some limits, for example, being single-threaded. Depending on what you're doing, you might be better off with something like python for smaller projects, maybe C# or Java for larger ones.

<a name="fn2">2:</a> Oof though, that `ui.R` file and its `shinyUI()` call -- you can really see the influence Lisp had on the original R/S syntax. 
