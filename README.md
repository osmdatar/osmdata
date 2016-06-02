[![Build Status](https://travis-ci.org/osmdatar/osmdatar.svg?branch=master)](https://travis-ci.org/osmdatar/osmdatar) [![codecov](https://codecov.io/gh/osmdatar/osmdatar/branch/master/graph/badge.svg)](https://codecov.io/gh/osmdatar/osmdatar)

![](./figure/map.png)

R package for downloading OSM data and converting to `sp` objects *really quickly*!

------------------------------------------------------------------------

Speed comparisons
-----------------

Speed comparisons can be examined in the branch `speed-comparisons`, with the main branch now just serving the further development of the `Rcpp` versions. The results from that branch (on test data of highways only) are:

| method                     | Computation time (s) |
|----------------------------|----------------------|
| osmplotr                   | 1.86                 |
| hrbrmstr                   | 1.47                 |
| Rcpp (-&gt;`sp` in `R`)    | 0.25                 |
| Rcpp (-&gt;`sp` in `Rcpp`) | 0.08                 |

Processing everything, including the construction of the `sp` S4 objects, within `Rcpp` routines is obviously enourmously faster (&gt;20 times). `osmdatar` downloads OSM data within `R` using `httr`, then passes the raw character file result to `Rcpp` routines which parse the XML structure and convert the results to S4 `sp` objects prior to return to `R`.

(See this [gist](https://gist.github.com/mpadge/c046031989e5eb7d8d8ec49a3a3136ae) for further exploration of ways to speed up construction of `sp` objects in `C++`.)

The package currently downloads and converts points, lines, and polygons, with the three respective functions:

1.  `get_nodes`

2.  `get_ways`

3.  `get_polygons`

The only remaining task is to implement the conversion of OSM `multipolygon` objects.

------------------------------------------------------------------------

Install
-------

``` r
devtools::install_github ('osmdatar/osmdatar')
```

------------------------------------------------------------------------

Running Speed Tests
-------------------

The speed of `osmdatar` can be compared with `osmplotr`, which uses `osmar` to download and process OSM data. Raw data are downloaded directly from the [overpass API](https://overpass-api.de) and can be downloaded with `osmdatar` with this code:

``` r
bbox <- matrix (c (-0.12, 51.51, -0.11, 51.52), nrow=2, ncol=2) 
dat_raw <- get_ways (bbox=bbox, raw_data=TRUE, key='highway')
```

This returns a character string representing the (non-parsed) XML data.

------------------------------------------------------------------------

### `osmplotr`

`osmplotr` has a single function (`extract_osm_data`) that both downloads and processes OSM data. To time just the processing stage, the portion of the code in which the download is processed has to be re-created here (hard-coded for `key=highway`):

``` r
library (osmplotr)
```

``` r
process_osmar <- function (dat)
{
    dat <- XML::xmlParse (dat)
    dato <- osmar::as_osmar (dat)
    for (i in seq (dato$relations))
        if (nrow (dato$relations [[i]]) > 0)
            dato$relations [[i]]$id <- paste0 ('r', dato$relations [[i]]$id)
    pids <- osmar::find (dato, osmar::way (osmar::tags(k == 'highway')))
    pids1 <- osmar::find_down (dato, osmar::way (pids))
    pids2 <- osmar::find_up (dato, osmar::way (pids))
    pids <- mapply (c, pids1, pids2, simplify=FALSE)
    pids <- lapply (pids, function (i) unique (i))
    nvalid <- sum (sapply (pids, length))
    obj <- subset (dato, ids = pids)
    osmar::as_sp (obj, 'lines')
}
```

Then the timing:

``` r
library (sp) # required for the osmar conversion
library (microbenchmark)
mb <- microbenchmark ( obj <- process_osmar (dat_raw), times=100L )
tt <- formatC (mean (mb$time) / 1e9, format="f", digits=2)
```

``` r
cat ("Mean time to convert with osmar =", tt, "s\n")
```

    ## Mean time to convert with osmar = 1.86 s

------------------------------------------------------------------------

### `osmdatar`

`osmdatar` allows raw data to first be downloaded prior to re-submission to the same function for subsequent parsing. (This allows different elements to be extracted quickly and conveniently from a single download.) The `osmdatar` speed test thus simply requires:

``` r
mb <- microbenchmark ( obj <- get_ways (url_download=dat_raw), times=100L )
tt <- formatC (mean (mb$time) / 1e9, format="f", digits=2)
```

``` r
cat ("Mean time to convert with osmdatar =", tt, "s\n")
```

    ## Mean time to convert with osmdatar = 0.08 s
