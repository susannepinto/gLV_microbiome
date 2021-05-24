---
title: R code "Correlations cannot distinguish interaction types in microbial networks"
author: "Susanne Pinto, Elisa Benincà, Egbert H. van Nes, Marten Scheffer and Johannes A. Bogaards"
date: "22/05/2021"
output: 
  github_document:
    toc: true
    toc_depth: 3
editor_options:
  chunk_output_type: console
---

```{r setup, include = FALSE}
knitr::opts_chunk$set(
  echo = TRUE, message = FALSE, warning = FALSE, comment = NA, cache = TRUE,
  fig.width = 7, fig.height = 7)
```


# Contents
### 1- Introduction
### 2- Install and load the packages
### 3- The generalized Lotka-Volterra model
### 4- Choose your settings and draw random parameters
### 5- Solve the model
### 6- The data
### 7- Calculate the partial correlations
### 8- Calculate the precision, recall and F1 score



## 1- Introduction
R code belonging to the paper: “Correlations cannot distinguish interaction types in microbial networks” by Susanne Pinto, Elisa Benincà, Egbert H. van Nes, Marten Scheffer and Johannes A. Bogaards. This document shows the script used to perform the base case study of the paper.

We assessed how correlations between bacterial groups are shaped by their ecological interactions, and how correlations can be employed to signify these interactions. We specifically investigate how inference of microbial interactions is affected by interindividual variation in process parameters, and mainly focus on the necessary model assumptions and the meaning of the obtained correlation-based networks rather than underlying network properties. We use the generalized Lotka-Volterra (gLV) model to demonstrate the performance of correlation-based network reconstruction.


## 2- Install and load the packages
Install and load the packages in R.

```{r install and load the packages, echo = TRUE, results = FALSE}
if ( !"tidyverse" %in% .packages( all = TRUE ) ) install.packages( "tidyverse" )
library( tidyverse )

if ( !"deSolve" %in% .packages( all = TRUE ) ) install.packages( "deSolve" )
library( deSolve )

if ( !"seqtime" %in% .packages( all = TRUE ) ) install.packages( "seqtime" )
library( seqtime ) 

if ( !"purrr" %in% .packages( all = TRUE ) ) install.packages( "purrr" )
library( purrr )

if ( !"ggplot2" %in% .packages( all = TRUE ) ) install.packages( "ggplot2" )
library( ggplot2 )

if ( !"reshape2" %in% .packages( all = TRUE ) ) install.packages( "reshape2" )
library( reshape2 )

if ( !"truncnorm" %in% .packages( all = TRUE ) ) install.packages( "truncnorm" )
library( truncnorm )

if ( !"Matrix" %in% .packages( all = TRUE ) ) install.packages( "Matrix" )
library( Matrix )

```



## 3- The generalized Lotka-Volterra model
We simulated bacterial communities by means of the generalized Lotka-Volterra (gLV) model. We simulated the population growth of species i with intrinsic growth rates (r), carrying capacities (K) and interactions with other species (a) that are specific for species i as well as for host m (see methods).


```{r the model, echo = TRUE}
gLV.model <- function( time, y, parms ) { 
  # change the parms to the .noise version if desirable  
  r_im = parms$r_im %>% as.numeric()
  K_im = parms$K_im %>% as.numeric() 
  a_ijm = parms$a_ijm %>% as.matrix() 
  
  # the function
  dydt <- r_im * y * ( 1 - ( 1/K_im ) * y + ( a_ijm %*% y ))
  
  # store data in a list
  dydt %>% list() 
} 
```



## 4- Choose your settings and draw random parameters
First set some standard settings. Optionally you can choose a higher number of species and samples. 

```{r settings, echo = TRUE}
# Set date / time of the simulation  
# This is used in the filenames
Date <- Sys.time() %>% format( ., format="Date %d-%m-%Y_Time %H.%M" )

# Choose the number of simulations
# In the paper, 1000 simulations were used, here we choose 100 simulations because it is more time efficient
n.simulations = 100

# Size of the population / total number of species
n.species = 10

# Number of samples (people)
n.samples = 300
# Later the samples with species that go extinct are removed, therefor simulate more samples than preferred 
n.samples.total = 400

# Timescale
tstart = 0
tend = 100
tstep = 1
times <- seq( from = tstart, to = tend, by = tstep ) 
```

Set (and save) your seeds for reproducibility.

```{r set seeds, echo = TRUE}
# Sample integers to later set as seed
seed <- 100000 %>% sample.int( ., n.simulations, replace = FALSE )

# save the seed list
seed %>% saveRDS( object = ., file = paste0( "seeds_", Date, ".Rds" ) )
```

With the following script the growth rates, carrying capacities and the interaction matrix are computed randomly from a given distribution.

```{r random parameters, echo = TRUE}
# make a list of n.simulations
parms <- n.simulations %>% vector( mode = "list", length = . )

# Do the simulation n.simulation * n.sample times
for ( j in 1:length( parms ) ) {
  # make a list in the list of length N.samples
  parms[[j]] <- n.samples.total %>% vector( mode = "list", length = . )
  
  for ( k in 1:n.samples.total ) {
    
    # use the saved seeds
    seed[j] %>% set.seed()
    
    # Set random parameters

    # initial abundances per species
    parms[[j]][[k]]$yini = n.species %>% runif( ., 0, 1 ) 
    # Intrinsic growth rates per species
    # Keep them positive to make all the species able to grow on their own
    # In the paper the growth rates were smaller, here we choose bigger values to decrease simulation time (e.g. species reach their equilibrium sooner with bigger growth rates)
    parms[[j]][[k]]$r_i = n.species %>% runif( ., 0.5, 1 ) 
    # carrying capacities per species
    parms[[j]][[k]]$K_i = n.species %>% runif( ., 0, 1 ) 
    
    # First draw the interactions (a_ij) than fill the interaction matrix
    # Interactions are drawn from two truncated normal distributions (which results in a bimodal distribution)
    negative_interactions <- ( ( n.species * n.species )/ 2 ) %>% 
      rtruncnorm( n = ., a = -0.5, b = 0.15, mean = -0.25, sd = 0.1 )
    positive_interactions <- ( ( n.species * n.species )/ 2 ) %>% 
      rtruncnorm( n = ., a = -0.15, b = 0.5, mean = 0.25, sd = 0.1 )
    
    # Determine the sparcity
    # The lower the last value the sparcer the matrix will be
    zero_index.negative_interactions <- sample( length( negative_interactions ) )[ 1:round(
      length( negative_interactions ) / 1.25 ) ]
    zero_index.positive_interactions <- sample( length( positive_interactions ) )[ 1:round(
      length( positive_interactions ) / 1.25 ) ]
    
    # Remove the values determined by the zero_index
    negative_interactions[ zero_index.negative_interactions ] <- 0
    positive_interactions[ zero_index.positive_interactions ] <- 0
    
    # Remove n.species/2 number of 0s because of diagonal a.matrix
    negative_interactions <- negative_interactions[ -sample( which( negative_interactions == 0
                                                                    ), n.species/ 2 ) ]
    positive_interactions <- positive_interactions[ -sample( which( positive_interactions == 0
                                                                    ), n.species/ 2 ) ]
    
    # Combine postive and negative interactions into a bomodal distribution
    data.all <- c(negative_interactions, positive_interactions)
    # Randomize order
    data.all <- data.all %>% sample
    
    # Make an empty a matrix with diagonal 0 
    a.matrix <- matrix( NA, ncol = n.species, nrow = n.species )
    diag( a.matrix ) = 0
    
    # Fill in the matrix (but not the diagonal)
    a.matrix <- data.all %>% ifelse( is.na( a.matrix ), ., 0 )

    # Set the ro- and colnames
    rownames( a.matrix ) <- "Species." %>% paste0( ., 1:n.species )
    colnames( a.matrix ) <- "Species." %>% paste0( ., 1:n.species )
    
    parms[[j]][[k]]$a.matrix <- a.matrix
    
  }
}
```

Visualization one of the interaction matrices

```{r visualize matrix, echo = TRUE}
# visualise one interaction matrix
a.matrix_sparse <- parms[[1]][[1]]$a.matrix %>% Matrix(., sparse=TRUE)
a.matrix_sparse %>% image()
```

We added multiplicative noise to the parameters to make them host specific. Here only the interspecies interactions are host specific, but it is possible to make the r_i and K_i also host-specific.

```{r multiplicative noise, echo = TRUE}
for ( l in 1:length( parms ) ) {
  for ( m in 1:n.samples.total ) {
    # Add noise to the interaction matrix
    noise.on.a_ij <- parms[[l]][[m]]$a.matrix %>% length() %>% rnorm( ., 0, 0.25 )
    # Only add noise when the value in the a.matrix is not 0
    parms[[l]][[m]]$a_ijm <- ifelse( parms[[l]][[m]]$a.matrix != 0, parms[[l]][[m]]$a.matrix * exp( noise.on.a_ij ), 0 )
    
    # Add noise to the growth rates
    noise.on.r_i <- n.species %>% rnorm( ., 0, 0 )
    parms[[l]][[m]]$r_im <- parms[[l]][[m]]$r_i * exp( noise.on.r_i )
    
    # Add noise to the carrying capacities
    noise.on.K_i <- n.species %>% rnorm( ., 0, 0 )
    parms[[l]][[m]]$K_im <- parms[[l]][[m]]$K_i * exp( noise.on.K_i )
  }
}

# Save the parms list
parms %>% saveRDS( object = ., file = paste0( "parms_original_data_", Date, ".Rds" ) )
```



## 5- Solve the model
The gLV model was solved with the lsoda function (from the deSolve package version 1.24). 

```{r solving the model, echo = TRUE, results = "hide" }
# Make a list of n.simulations
community.times <- n.simulations %>% vector( mode = "list", length = . )

# Do for every simulation
for ( n in 1:length( community.times ) ) {
  # Make a list in the list of length N.samples
  community.times[[n]] <- n.samples.total %>% vector( mode = "list", length = . )
  
  tryCatch( {
    
    # Do for every sample / individual
    for ( o in 1:length( community.times[[n]] ) ) {
      
      # Random yini than yini = N.species.total %>% runif()
      community.time = lsoda( y = parms[[n]][[o]][ c( "yini" ) ] %>% unlist %>% as.numeric, 
                              times = times, 
                              func = gLV.model, 
                              parms = parms[[n]][[o]] )
      
      # First column of community time dataset is time
      time = community.time[,1] 
      # Second column till the last species is results oda
      community.time = community.time[, 2:ncol( community.time ) ] 
      # Transpose table
      community.time = t( community.time ) 
      # use times as colnames
      colnames( community.time ) = times
      
      # Store the time series in the community times list
      community.times[[n]][[o]]$community.time <- community.time
    }
    # Do not stop the simulation because of an error or warning (but do not return any results)
  }, warning = function( e ){ return( NULL ) } )
  # Print number of simulation, to see progress
  n %>% print()
}
```



Choose a community and plot to see the time series.

```{r plot the community, echo = TRUE}
# pick one community
choose.community <- community.times[[1]][[1]] %>% as.data.frame()
# show just one figure
par( mfrow = c( 1, 1 ) )
# make a time plot (from the package seqtime)
choose.community %>% tsplot( ., legend = F )
```



## 6- The data
Most gut microbiota studies are limited to only a few samples in time, presenting mere ‘snapshots’ of the intestinal ecosystem. 

```{r Snapshot, echo = TRUE  }
# Make a list of n.simulations
community.samples <- n.simulations %>% vector( mode = "list", length = . )

# Do for every simulation, sample at the last timepoint
for ( q in 1:length( community.samples ) ) {
  # Create an empty data frame to store the results
  community.samples[[q]] <- matrix( NA, nrow = n.species, ncol = n.samples.total )
  
  tryCatch({  
    # Do for every sample / individual
    for (r in 1:length( community.times[[q]] ) ) {
      
      # Change the data to a data frame
      abundances <- community.times[[q]][[r]] %>% as.data.frame()
      # Get the name of the last column
      keep <- c( tend + 1 )
      # Keep only the abundances at the last time point (preferably  in the equilibrium)
      abundances <- abundances[keep] %>% simplify2array()
      
      # Store the abundances on the last time point in the list
      community.samples[[q]][,r] <- abundances
      
    }
  }, error = function( e ){ return( NULL ) } )
}
```

We also added additive noise, drawn from a uniform distribution, to the outcome of the gLV model to represent uncertainty in measurements.

```{r measurement noise, echo = TRUE  }
# Make a list of n.simulations
community.samples.withnoise <- n.simulations %>% vector( mode = "list", length = . )

# Do for every simulation
for (s in 1:length( community.samples.withnoise )) {
  # Create noise values
  measurement.noise <- community.samples[[s]] %>% length() %>% 
    runif( ., -0.01, 0.01 ) %>% 
    matrix( ., nrow = n.species, ncol = n.samples.total )
  # Add additive noise to the dataset
  community.samples.withnoise[[s]] <- community.samples[[s]] + measurement.noise
  
  # Create col and row names
  nms.of.samples = "Sample." %>% paste0( ., 1:n.samples.total )
  nms.of.species = parms[[1]][[1]]$a.matrix %>% colnames()
  # Rename columns and rows
  colnames( community.samples.withnoise[[s]] ) = nms.of.samples 
  rownames( community.samples.withnoise[[s]] ) = nms.of.species
  }
```

If species did not co-exist (when the abundance of a species drops below 0.001), we rejected the simulation. 

```{r remove species that got extinct, echo = TRUE  }
# Make a list of n.simulations
community.samples.survival <- n.simulations %>% vector( mode = "list", length = . )

for ( t in 1:length( community.samples.survival ) ) {
  # Transpose the dataset
  community.samples.transpose <- community.samples.withnoise[[t]] %>% t()
  # Change the values of species with an abundance below 0.01 to NA
  community.samples.survive <- ifelse( community.samples.transpose < 0.001, NA, 
                                       community.samples.transpose )
  community.samples.survive <- community.samples.survive %>%  as.data.frame()
  # Remove columns with all NAs
  community.samples.survive <- Filter( function( x ) !all( is.na( x ) ), community.samples.survive )
  # Remove rows with some NAs
  community.samples.survive <- community.samples.survive[complete.cases( community.samples.survive ), ] %>% data.frame()
  
  # Number of species left
  species.left <- ncol( community.samples.survive ) 
  # Number of samples left
  samples.left <- nrow( community.samples.survive ) 
  
  survival <- c( species.left, samples.left ) %>% print()
  
  # Remove the simulation if not enough species and samples are left
  if ( species.left < n.species  | samples.left < n.samples ) {
    print( "not enough species survived" )
  } else {
    community.samples.survival[[t]]$data <- community.samples.survive %>% as.matrix()
  }
}
```

```{r remove samples with not enough species, echo = TRUE }
# check if there are simulations with not enough species or samples
removed.simulations.survival_error <- n.simulations %>% character()
for ( u in 1:length( community.samples.survival ) ) {
  # if errors occured, print number
  if( length( unlist( community.samples.survival[[u]] ) ) == 0 ) {
    # store the numbers of the simulations with errors
    removed.simulations.survival_error[[u]] <- print( u )
  } else {
    removed.simulations.survival_error[[u]] <-print( NA )
  }
}

# remove empty list elements
community.samples.survival <- community.samples.survival %>% compact()
```


The samples that are removed in the previous step, also needs to be removed from the parms data.

```{r remove samples and species, echo = TRUE, error = TRUE  }

# Removed simulations after extinction of species
removed.simulations <- removed.simulations.survival_error %>% as.numeric()

# Save only the parms that were not removed (== NA) 
for (v in 1:length( parms )) {
  if ( removed.simulations[[v]]  %>% is.na() ) {
    parms[[v]] <- parms[[v]]
  }   else {
    parms[[v]] <- list( NULL )
  }
}

# Remove the simulations where errors or not enougs samples/species survived have occurred
parms <- Filter( function( x ) any( unlist( x ) != 0 ), parms )

# Save the new parms list
parms %>% saveRDS( object = ., file = paste0( "parms_data_", Date, ".Rds" ) )
```

We choose a higher number of samples than the preferred number we want to end up with. Therefore, randomly remove some of the samples to end up with the preferred number of samples in which all species coexist.

```{r update parms, echo = TRUE}
# Make a list of n.simulations
data <- length( community.samples.survival ) %>% vector( mode = "list", length = . )

# Keep randomly the preferred number of samples
for ( w in 1:length( data ) ) {
  # Number of samples left
  samples.left <- nrow( community.samples.survival[[w]][["data"]] ) 
  # Get the number of samples that needs to be removed
  n.samples.to.remove = samples.left - n.samples
  # Transpose the data and save as data.frame
  data.samples <- community.samples.survival[[w]][["data"]] %>% t() %>% as.data.frame()
  # Choose the samples that will be removed
  samples.to.remove <- data.samples %>% 
    sample( ., n.samples.to.remove )
  # Get the names of the samples to be removed
  names.samples.to.remove <- samples.to.remove %>% colnames()
  
  # Select the samples to keep
  data[[w]]$data <- data.samples %>% 
    select( ., -one_of( names.samples.to.remove ) )
}

# Transform the data
for ( x in 1:length( data ) ) {
  data[[x]]$data <- data[[x]][["data"]] %>% t() %>% as.data.frame()
}

# Number of species
data[[1]][["data"]] %>% ncol()
# Number of samples
data[[1]][["data"]] %>% nrow()

# Save the parms list
data %>% saveRDS( object = ., file = paste0( "Final_data_", Date, ".Rds" ) )

# Make a list of n.simulations
names <- length( data ) %>% vector( mode = "list", length = . )

# Do for every sample / individual
for ( y in 1:length( data ) ) {
  names[[y]][["Samples"]] <- rownames( data[[y]][["data"]] )
  names[[y]][["Species"]] <- colnames( data[[y]][["data"]] )
}

# Save a list with all species names
names %>% saveRDS( object = ., file = paste0("names_data_", Date, ".Rds" ) )

```


## 7- Calculate the partial correlations
First, we calculate the partial correlations and save these.

```{r partial correlations, echo = TRUE  }
# Make a list of length  data
results.pcor <- length( data ) %>% vector( mode = "list", length = . )

for ( i in 1:length( data ) ) {
  # Calculate the covariance matrices from the abundance tables
  cx <- data[[i]][["data"]] %>% as.data.frame() %>% cov()
  
  # Calculate the inverse of the covariance matrix
  dx <- cx %>% solve( ., sparse = TRUE,  tol = 1e-150 )
  # Calculate the partial correlation matrix
  pcor <- -cov2cor( dx )
  
  # Change the diagonal to 1 (instead of -1)
  diag( pcor ) <- 1 
  
  # Store the results in the list
  results.pcor[[i]]$pcor <- pcor %>% as.matrix()
}

# Save the results list
results.pcor %>% saveRDS( object = ., file = paste0( "Partial_correlations_", Date, "_Results.Rds" ) )
```

Hereafter we should keep only the significant partial correlations. These significant correlation matrices are transformed to a matrix where, negative correlations are assigned a "-1", positive correlations are assigned a "1" and if a correlations was not significant than the value "0" is implemented.

```{r significant correlations, echo = TRUE  }
# Do for all the pcors
pcor.pvalues.significant <- length( results.pcor ) %>% vector( mode = "list", length = . )

for ( i in 1:length( results.pcor ) ) {
  # Number of samples
  pcor.n <- dim( data[[i]][["data"]] )[1]
  pcor.gp <- dim( data[[i]][["data"]] )[2]-2
  
  # Do the statistics
  pcor.statistic <- results.pcor[[i]][["pcor"]] * 
    sqrt( ( pcor.n-2-pcor.gp ) / ( 1-results.pcor[[i]][["pcor"]]^2 ) )
  # Get the p values
  pcor.pvalues <- 2 * pnorm( -abs( pcor.statistic ) )
  
  # Adjust the p-values with Benjamini Hochberg procedure
  pcor.pvalues.adjusted <- array(NA, dim=dim(pcor.pvalues))
  names <- colnames(pcor.pvalues)
  colnames(pcor.pvalues.adjusted) <- names
  rownames(pcor.pvalues.adjusted) <- names
  for (j in 1:dim(pcor.pvalues)[1])
    pcor.pvalues.adjusted[j,] <- p.adjust(pcor.pvalues[j,], method = "BH")
  
  # Keep the significant results
  pcor.pvalues.sign <- pcor.pvalues.adjusted < 0.05
  # Change true in 1 and false to 0
  pcor.pvalues.sign <- 1 * pcor.pvalues.sign
  
  # Change the matrices to negative (-1), no (0) and positive interaction (-1) metrices
  pcor.pvalues.sign <- ifelse( pcor.pvalues.sign == 0, 0,
                               ifelse( results.pcor[[i]][["pcor"]] >= 0, 1, -1 ) )
  
  # Remove the diagonal, because that has nothing to do with interactions
  diag( pcor.pvalues.sign ) <- NA
  
  # Store in the results
  pcor.pvalues.significant[[i]] <- pcor.pvalues.sign
}

# save the results list
pcor.pvalues.significant %>% saveRDS( object = ., file = paste0( "Partial_correlations_sign_",
                                                                 Date, "_Results.Rds" ) )
```



## 8- Calculate precision, recall and the F1-score
These measures are used to compare the original interaction matrix from the model with the inferred significant partial correlations. 

```{r prepare the data, echo = TRUE}
# Make a list 
a.matrix.analysis <- list() 

# Subtract all the interaction matrices
for ( i in 1:length( parms ) ) {
  # Make a list per simulation
  a.matrix.analysis[[i]] <- length( parms[[i]] ) %>% vector( mode = "list", length = . )
  for ( j in 1:length( parms[[i]] ) ) { 
    # Here the a matrix without noise is used
    a.matrix.analysis[[i]]<- parms[[i]][[j]][ c( "a.matrix" ) ] 
  }
}

# Flatten the list of lists
a.matrix.analysis <- a.matrix.analysis %>% flatten()

```

After selecting the variables of interest, the interaction matrix is transformed similar to the significant partial correlation matrix, with the -1, 0 and 1 for respectively negative, no and positive interactions.

```{r transform the interaction matrix, echo = TRUE}
# Do for the a.matrices
a.matrix.round <- length( a.matrix.analysis ) %>% vector( mode = "list", length = . )

for ( i in 1:length( a.matrix.round ) ) {
  # change the matrices to negative (-1), no (0) and positive interaction (-1) metrices
  a.matrix.analysis.01 <- ifelse( a.matrix.analysis[[i]] == 0, 0,
                                  ifelse( a.matrix.analysis[[i]] >= 0, 1, -1 ) )
  
  # Remove the diagonal, because that has nothing to do with interactions
  diag( a.matrix.analysis.01 ) <- NA
  
  # Store in the results
  a.matrix.round[[i]] <- a.matrix.analysis.01
}
```

Calculate the confusion matrix

```{r confusion matrix, echo = TRUE}
# Make a new list
confusion.matrix <- length(a.matrix.round) %>% vector( mode = "list", length = . )

# Do for every simulation
for ( i in 1:length( a.matrix.round ) ) {
  
  # Make an actual and predicted matrix
  actual <- a.matrix.round[[i]]
  predicted <- pcor.pvalues.significant[[i]]
  
  # Change the lower half of the predicted matrix to na (because it is symmetric)
  predicted[ lower.tri( predicted ) ] <- NA
  
  # Make a empty confusion matrix to store the results
  # Actual is rows, predicted is cols
  results <- matrix( 0, 3, 3 )
  colnames( results ) <- c( "-1","0", "1" )
  rownames( results ) <- c( "-1","0", "1" )
  
  # Go through every row and columns
  for ( r in 1:nrow( predicted ) ) {   
    for ( c in 1:ncol( predicted ) ) {
      
      # If the value is na, than it was a diagonal and do nothing
      if( is.na( predicted[r, c] ) ) {
        results[1, 1] = results[1, 1] + 0
        
        # If cor is 0 and interactions are both 0
      } else if ( predicted[r, c] == 0 & ( actual[r, c] == 0 & actual[c, r] == 0 ) ) {
        # Than a True negative
        results[2, 2] = results[2, 2] + 1
        # If cor is 0 and  one of the interactions interactions are 1
      } else if ( predicted[r, c] == 0 & ( actual[r, c] == 1 | actual[c, r] == 1 ) ) {
        # Than a False negative
        results[3, 2] = results[3, 2] + 1
        # If cor is 0 and  one of the interactions interactions are -1
      } else if ( predicted[r, c] == 0 & ( actual[r, c] == -1 | actual[c, r] == -1 ) ) {
        # Than a False negative
        results[1, 2] = results[1, 2] + 1
        
        # If cor is 1 and one of the interactions interactions are 1
      } else if ( predicted[r, c] == 1 & ( actual[r, c] == 1 | actual[c, r] == 1 ) ) {
        # Than a True positive
        results[3, 3] = results[3, 3] + 1
        # If cor is 1 and interactions are both 0
      } else if ( predicted[r, c] == 1 & ( actual[r, c] == 0 & actual[c, r] == 0 ) ) {
        # Than a False positive
        results[2, 3] = results[2, 3] + 1
        # If cor is 1 and interactions are both -1
      } else if ( predicted[r, c] == 1 & ( actual[r, c] == -1 & actual[c, r] == -1 ) ) {
        # Than a False positive (wrong sign)
        results[1, 3] = results[1, 3] + 1
        
        # If cor is -1 and one of the interactions are  -1
      } else if ( predicted[r, c] == -1 & ( actual[r, c] == -1 | actual[c, r] == -1 ) ) {
        # Than a True positive
        results[1, 1] = results[1, 1] + 1
        # If cor is -1 and interactions are both 0
      } else if ( predicted[r, c] == -1 & ( actual[r, c] == 0 & actual[c, r] == 0 ) ) {
        # Than a False positive
        results[2, 1] = results[2, 1] + 1
        # If cor is -1 and interactions are both 1
      } else if( predicted[r, c] == -1 & ( actual[r, c] == 1 & actual[c, r] == 1 ) ) {
        # Than a False positive (wrong sign)
        results[3, 1] = results[3, 1] + 1
      }
      # Store the results in the list
      confusion.matrix[[i]] <- results
    }
  }
}

# Save the confusion matrices list
confusion.matrix %>% saveRDS( object = ., file = paste0( "confusion.matrix_", Date, "_Results.Rds" ) )
```

Calculate the precision, recall and f1 score

```{r scores,  , echo = TRUE}
# Make a new list
F1.score <- length( confusion.matrix ) %>% vector( mode = "list", length = . )

# Calculate F1_score with paying attention to the sign
for ( i in 1:length( confusion.matrix ) ) {
  confusion <- confusion.matrix[[i]]
  
  # Calculate the precision
  precision = ( confusion[1, 1] + confusion[3, 3] ) / ( ( confusion[1, 1] + confusion[3, 3] ) + 
                                                          ( confusion[2, 1] + confusion[3, 1] + confusion[1, 3] + confusion[2, 3] ) )
  F1.score[[i]]$precision <- precision
  
  # Calculate the recall
  recall = ( confusion[1,1] + confusion[3,3] ) / ( ( confusion[1,1] + confusion[3,3] ) + 
                                                     ( confusion[1,2] + confusion[3,2] ) )
  F1.score[[i]]$recall <- recall
  
  # Calculate the F1 score
  F1score <- 2 * ( ( precision * recall ) / ( precision + recall ) )
  F1.score[[i]]$F1score <- F1score
}

# Save the precision,recall and F1scores list
F1.score %>% saveRDS( object = ., file = paste0("F1.score_", Date, "_Results.Rds" ) )
```

Plot the results 
```{r plot the results, echo = TRUE}
# Make an empty data frame
F1.score.data <- matrix( 0, nrow = length( F1.score ), ncol = 3 )

# Do for every simulation
# Make a dataframe with the results
for ( i in 1:length( F1.score ) ) {
  F1.score.data[i, 1] <- F1.score[[i]][[ "precision" ]]
  F1.score.data[i, 2] <- F1.score[[i]][[ "recall" ]]
  F1.score.data[i, 3] <- F1.score[[i]][[ "F1score" ]]
}

# Change the NA's in the data to 0
F1.score.data <- F1.score.data %>% replace( ., is.na( F1.score.data ), 0 ) 
# Name the columns
colnames( F1.score.data ) <- c( "Precision", " Recall", " F1.score" )

F1.score.data.2 <- F1.score.data %>% melt( ., id.vars = "F1.score" )
str( F1.score.data.2 )
F1.score.data.2$value <- as.numeric( as.character( F1.score.data.2$value ) )

# Make the boxplot
boxplot <- ggplot( data = F1.score.data.2, 
                   mapping = aes( x = Var2, y = value ) ) +
  # Make a boxplot
  geom_boxplot() +
  # Name the labels
  labs( x = "variable", y = "value" ) +
  # Text over an angle
  theme( axis.text.x = element_text( angle = 45, hjust = 1 ) )  + 
  # Color the boxes
  geom_boxplot(
    aes( fill = Var2 ),
    position = position_dodge( 0.9 ) ) + 
  ggtitle( paste( "Results", sep = ",", collapse = NULL) )  +
  scale_y_continuous( limits = c( 0, 1 ) )
# Plot the boxplot
boxplot
```
