
<!-- README.md is generated from README.Rmd. Please edit that file -->

# Post R plot to Instagram on RStudio (or on whatever editor)

## Install chromote

  - chromote: <https://github.com/rstudio/chromote>

<!-- end list -->

``` r
devtools::install_github("rstudio/chromote")
```

### (Linux only?) Set `CHROMOTE_CHROME`

I use Manjaro Linux, so I needed to set this by myself.

``` r
Sys.setenv(CHROMOTE_CHROME="/usr/bin/chromium")
```

### Confirm chromote works

``` r
library(chromote)

b <- ChromoteSession$new()
b$Browser$getVersion()
#> $protocolVersion
#> [1] "1.3"
#> 
#> $product
#> [1] "HeadlessChrome/80.0.3987.122"
#> 
#> $revision
#> [1] "@cf72c4c4f7db75bc3da689cd76513962d31c7b52"
#> 
#> $userAgent
#> [1] "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/80.0.3987.122 Safari/537.36"
#> 
#> $jsVersion
#> [1] "8.0.426.25"
```

## Get device profiles

The profiles of mobile devices are available on
`front_end/emulated_devices/module.json` of
[`ChromeDevTools/devtools-frontend`
repository](https://github.com/ChromeDevTools/devtools-frontend).

c.f. <https://github.com/mafredri/cdp/issues/93#issuecomment-486683596>

``` r
dir.create("data", showWarnings = FALSE)
download.file(
  "https://raw.githubusercontent.com/ChromeDevTools/devtools-frontend/master/front_end/emulated_devices/module.json",
  destfile = "data/module.json"
)
```

``` r
library(purrr)

j <- jsonlite::read_json("data/module.json")

idx <- map_chr(j$extensions, "type") == "emulated-device"
j <- map(j$extensions, "device")[idx]
names(j) <- map_chr(j, "title")
```

## Set up emulation

``` r
device <- j$`Nexus 5X`
orientation <- "vertical" # can choose vertical or horizontal

b$Emulation$setUserAgentOverride(userAgent = device$`user-agent`)
#> named list()
b$Emulation$setDeviceMetricsOverride(
  deviceScaleFactor = device$screen$`device-pixel-ratio`,
  width = device$screen[[orientation]]$width,
  height = device$screen[[orientation]]$height,
  mobile = TRUE
)
#> named list()
```

## Login

``` r
b$Page$navigate("https://www.instagram.com/accounts/login/")

b$view()
# (Manually log in)

cookies <- b$Network$getCookies()
saveRDS(cookies, "data/cookies.rds")
```

``` r
cookies <- readRDS("data/cookies.rds")
b$Network$setCookies(cookies = cookies$cookies)
#> named list()

b$Page$navigate("https://example.com/")
#> $frameId
#> [1] "D920E371A4CC169BD4DE4399827699ED"
#> 
#> $loaderId
#> [1] "AAE92BFD04149CE214F9698295DC248C"

p <- b$Page$navigate("https://www.instagram.com/", wait_ = FALSE)$
  then(function(value) {
    b$Page$loadEventFired(wait_ = FALSE)
  })

b$wait_for(p)
#> $timestamp
#> [1] 5847.551

screenshot_tmp <- tempfile(fileext = ".png")
b$screenshot(filename = screenshot_tmp)
#> [1] "/tmp/RtmpqgWpq5/file245d604c4f4.png"

knitr::include_graphics(screenshot_tmp)
```

<img src="/tmp/RtmpqgWpq5/file245d604c4f4.png" width="300px" />

## Post a plot

(This chunk is not executed because I don’t want to post the same plot
thousands of times…)

``` r
library(ggplot2)

p <- ggplot(mtcars, aes(factor(cyl), mpg)) +
  geom_violin(aes(fill = factor(cyl)))
tmp <- tempfile(fileext = ".jpg")
ggsave(p, filename = tmp)

# content is a Quad object:
# "An array of quad vertices, x immediately followed by y for each point, points clock-wise."
calc_center_of_content <- function(content) {
  list(
    x = (content[[1]] + content[[3]]) / 2,
    y = (content[[2]] + content[[8]]) / 2
  )
}

# insert file
# need to click + button to pretend as a human
root <- b$DOM$getDocument()$root$nodeId
divs <- b$DOM$querySelectorAll(root, "nav div")
is_plus <- map_lgl(divs$nodeIds, ~ "new-post-button" %in% b$DOM$getAttributes(.)$attributes)
plus_button <- b$DOM$getBoxModel(divs$nodeIds[[which(is_plus)]])
ctr <- calc_center_of_content(plus_button$model$content)
b$Input$synthesizeTapGesture(x = ctr$x, y = ctr$y)

Sys.sleep(1)

root <- b$DOM$getDocument()$root$nodeId
file_inputs <- b$DOM$querySelectorAll(root, "form input")
length(file_inputs$nodeIds)
b$DOM$setFileInputFiles(
  list(tmp),
  file_inputs$nodeIds[[length(file_inputs$nodeIds)]]
)

Sys.sleep(1)

# tap "Next"
root <- b$DOM$getDocument()$root$nodeId
buttons <- b$DOM$querySelectorAll(root, "button")
is_next <- map_lgl(buttons$nodeIds, ~ stringr::str_detect(b$DOM$getOuterHTML(.), "Next"))
button <- b$DOM$getBoxModel(buttons$nodeIds[[which(is_next)]])
ctr <- calc_center_of_content(button$model$content)
b$Input$synthesizeTapGesture(x = ctr$x, y = ctr$y)

Sys.sleep(1)

# Add text
root <- b$DOM$getDocument()$root$nodeId
textareas <- b$DOM$querySelectorAll(root, "textarea")
is_caption <- map_lgl(textareas$nodeIds, ~ "Write a caption…" %in% b$DOM$getAttributes(.)$attributes)
caption <- b$DOM$getBoxModel(textareas$nodeIds[[which(is_caption)]])
# move focus to text area
ctr <- calc_center_of_content(caption$model$content)
b$Input$synthesizeTapGesture(x = ctr$x, y = ctr$y)
# insert text
b$Input$insertText("This post is posted from RStudio")

Sys.sleep(1)

# tap "Share"
root <- b$DOM$getDocument()$root$nodeId
buttons <- b$DOM$querySelectorAll(root, "button")
is_share <- map_lgl(buttons$nodeIds, ~ stringr::str_detect(b$DOM$getOuterHTML(.), "Share"))
button <- b$DOM$getBoxModel(buttons$nodeIds[[which(is_share)]])
ctr <- calc_center_of_content(button$model$content)
b$Input$synthesizeTapGesture(x = ctr$x, y = ctr$y)
```

## End

``` r
b$close()
#> [1] TRUE
```
