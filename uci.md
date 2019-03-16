---
title: "Sensitivity analysis for Instrumental Variables models"
author: "Prodyumna Goutam"
date: "1/29/2019"
output: 
  html_document:
    keep_md: true
    toc: true
    toc_float: true
---

## Introduction 

This notebook explores sensitivity analysis for instrumental variables (IV) analysis. It is based on [this](https://www.mitpressjournals.org/doi/10.1162/REST_a_00139) paper by Conley and co-authors and the excellent stata package `plausexog` written by Damian Clarke and Benjamin Matta (with accompanying [paper](https://www.stata-journal.com/article.html?article=st0538))   

The basic approach can be stated as follows: all IV models critically rely on the **exclusion restriction** i.e. the instrument only affects the outcome through the endogenous variable of interest. In conducting analysis using observational data, we often run into situations where such a strong restriction is unlikely to hold. Conley et al. (2012) show approaches to conducting inference under relaxations of this requirement. 

To put things a little more formally and borrowing notation from the Conley paper, we have: 
\[
\begin{eqnarray}
  X & = & Z\Pi + V\\
  Y & = & X\beta + Z\gamma + \varepsilon\\
\end{eqnarray}
\]

The first equation is the usual first-stage relationship between the endogenous regression $X$ and the instrument $Z$. The second equation is the usual structural relationship between the outcome $Y$ and endogenous regressor $X$, with the presence of the instrument of the instrument $Z$. The usual exclusion restriction translates to $\gamma\equiv0$. 

But what if we were to relax that and assume a violation of the exclusion restriction? This translates to $\gamma\neq0$ and the term $Z\gamma$ does not drop out of the structural equation. What this means is that our instrument now has a direct effect on our outcome variable. The next section outlines one inference approach presented by Conley and co-authors to deal with this.   

## Union of Confidence Intervals


```r
library(AER)
library(foreign)
library(Formula)
library(formula.tools)
```



```r
uci <- function(form,instr,gmin,gmax,grid = 2,data) {
  # Check presence of model inputs 
  if (missing(form)) stop("Missing formula")
  if (missing(instr)) stop("Missing instruments")
  if (missing(gmin)) stop("Missing min support")
  if (missing(gmax)) stop("Missing max support")
  if (missing(data)) stop("Data not provided")
  
  # Check if size of support vectors match up 
  if (length(gmin) != length(gmax)) stop("Min and max lengths different")
  
  # Create support grid 
  gam_list <- list() 
  for (i in 1:length(gmin)){
    gam_list[[i]] <-seq(from = gmin[1], to = gmax[i], length.out = grid)
  }
  gamdf <- expand.grid(gam_list) #All combos of gamma values
  
  #Create list of modified IV formulae
  ivlist <- create_ivformula(form,instr,gamdf)
  
  # Iterate to get bounds 
  uci_bds <- uci_iterate(ivlist,data)
}
```


```r
create_ivformula <- function(form,instr,gamdf) {
  # Extract relevant formula components 
  iv <- formula.tools::rhs.vars(instr)
  endo <- formula.tools::lhs.vars(instr)
  dep <- formula.tools::lhs.vars(form)
  exo <- formula.tools::rhs.vars(form)
  
  # Underidentified model? 
  if(length(iv) < length(endo)) stop("Num iv less than num endo vars")
  
  # Construct RHS side of the formula 
  rhs1 <- paste(paste(exo, collapse = "+"),paste(endo, collapse = "+"),sep = "+") 
  rhs2 <- paste(exo <- paste(exo, collapse = "+"),paste(iv, collapse = "+"),sep = "+")
  
  # Create list of ivformula 
  ivlist <- list()
  for (i in 1:nrow(gamdf)) {
    gam <- unlist(gamdf[i,])
    dep_mod <- paste(gam,iv,sep = "*",collapse = "-")
    lhs <- paste0("I(",dep,"-",dep_mod,")")
    ivlist[[i]] <- as.Formula(paste(lhs,
                           paste(rhs1,rhs2, sep = "|"),
                           sep = "~"))
  }
  return(ivlist)
}
```


```r
uci_iterate <- function(ivlist,data) {
    iv_res <- lapply(ivlist,ivreg,data = data)
    ci <- lapply(iv_res,confint)
    # Construct lower and upped bound
    ci_l <- ci[[1]][,1]
    ci_h <- ci[[1]][,2]
    for (i in 2:length(ci)) {
      ci_l <- pmin(ci_l,ci[[i]][,1])
      ci_h <- pmax(ci_h,ci[[i]][,2])
    }
    uci_bounds <- cbind("lb" = ci_l,"ub" = ci_h)
    return(uci_bounds)
}
```


