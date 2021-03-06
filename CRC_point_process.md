Simulated examples
==================

Parameter estimation for the Thomas, Matérn, and void process is carried
out by the R package **palm** (B. C. Stevenson (2017)), which is
available on CRAN. All models are fitted using either the **fit.ns()**
function, for NSPP, or the **fit.void()** function, for the void
process.

Below, code examples demonstrate using functionality of the **palm** (B.
C. Stevenson (2017)) package to simulate from and fit a two-dimensional
point process. This is done for of each of the types mentioned above.
The simulated data for each process are shown---parents are plotted at
grey crosses and daughters by black dots---along with the fitted (solid
lines) and empirical (dashed lines) Palm intensity functions.

Please note that parameters of the Palm intensity functions may differ,
in name only, from the descriptions in Jones-Todd et al. (2017).

    library(palm)

Neyman-Scott point processes
----------------------------

Cluster processes where unobserved parent points randomly generate
daughters centered at their unobserved location. The spatial
distribution of daughters may eiher follow a bivariate normal
distribution (Thomas) or be uniformally distributed in a circle
(Mat'ern).

### Thomas process

The Palm intensity function of a Thomas process may be characterised by
the parameter vector **θ** = (*D*, *λ*, *σ*). Here *D* is the density of
parents, *λ* gives the expected number of daughters per parent, and *σ*
is the standard deviation of the bivariate normal distribution by which
daughters are scattered around their parents.

    set.seed(1234)
    lims <- rbind(c(0, 1),c(0,1)) ## 2D limits of domain (i.e., unit square)
    thomas <- sim.ns(c(D = 7, lambda = 8, sigma = 0.05), lims = lims) ## simulate a Thomas process
    fit.thomas <- fit.ns(thomas$points,lims = lims, R = 0.5) ## fit a Thomas process
    coef(fit.thomas)

    ##          D     lambda      sigma 
    ## 3.87798778 6.29007212 0.05679659

![](CRC_point_process_files/figure-markdown_strict/plot%20thomas-1.png)

### Matérn process

The Palm intensity function of a Matérn process may be characterised by
the parameter vector **θ** = (*D*, *λ*, *τ*). Here *D* is the density of
parents, *λ* gives the expected number of daughters per parent, and *τ*
is the radius of the sphere, centered at a parent location, within which
daughters are uniformally scattered.

    set.seed(2344)
    lims <- rbind(c(0, 1),c(0,1)) ## 2D limits of domain (i.e., unit square)
    matern <- sim.ns(c(D = 7, lambda = 8, tau = 0.05), lims = lims,disp = "uniform") ## simulate a Matern process
    fit.matern <- fit.ns(matern$points,lims = lims, R = 0.5, disp = "uniform") ## fit a Matern process
    coef(fit.matern)

    ##          D     lambda        tau 
    ## 5.43910354 5.05162971 0.04685375

![](CRC_point_process_files/figure-markdown_strict/plot%20matern-1.png)

Void point process
------------------

The Palm intensity function of a void process may be characterised by
the parameter vector
**θ** = (*D*<sub>*c*</sub>, *D*<sub>*p*</sub>, *τ*). Here
*D*<sub>*c*</sub> is the density of children, *D*<sub>*p*</sub> is the
density of parents, and *τ* is the radius of the voids centered at each
parent.

    set.seed(3454)
    lims <- rbind(c(0, 1),c(0,1)) ## 2D limits of domain (i.e., unit square)
    void <- sim.void(pars = c(Dc = 300, Dp = 10,tau = 0.075),lims = lims) ## simulate a void process
    fit.void <- fit.void(points = void$points, lims = lims, R = 0.5,
                         bounds = list(Dc = c(280,320), Dp = c(8,12), tau = c(0,0.2))) ## fit a void process
    coef(fit.void)

    ##           Dp           Dc          tau 
    ##   8.54355310 306.63597598   0.09059584

![](CRC_point_process_files/figure-markdown_strict/plot%20void-1.png)

Variance estimation
-------------------

Variance estimation is achieved via a parametric bootstrap, which can be
carried out through the use of the **boot.palm()** function. For
example, running **boot.palm(fit.thomas,N = 1000)** will perform 1000
bootstrap resamples of the fitted thomas process in order to estimate
standard errors of the parameters.

CRC data
========

As per Jones-Todd et al. (2017) below is an digital image of a tissue
section from a CRC patient (left hand plot of the Figure), the tumour
and stroma structures of the tissue sections are coloured in red and
green respectively. The far right plot of shows the point pattern formed
by the tumour and stroma cell nuclei, black and grey respectively, of
the same tissue section.

![Illustration of the image analysis of one patient's slide which
enables the pinpointing of nuclei. Left: Composite immunofluorescence
digital image showing Tumour (red), Stroma (green) and all nuclei
(blue). Middle: Image analysis mask overlay from automatic machine
learnt segmentation of the digital image. Tumour (purple), stroma
(turquoise), necrosis (yellow). Right: Point pattern formed by the
nuclei of the tumour (black) and stroma (grey) cells shown in the
previous two
images.](CRC_point_process_files/figure-markdown_strict/cancer.png)

    library(palm)
    ## to allow parallel computing from R
    library(foreach)
    library(doSNOW)
    R <- 0.5
    n <- length(Tu)
    ncores <- 5

    ## tumour locs
    points<-list()
    ## stroma locs
    pointss<-list()
    for(i in 1:n){
        points[[i]] <- cbind(Tu[[i]]$inner_x,Tu[[i]]$inner_y)/2500
        pointss[[i]] <- cbind(St[[i]]$inner_x,St[[i]]$inner_y)/2500
    }


    ## Set up results file
    ## ID, grades, and mortality index
    ID <- grade <- mort <- rep(NA,n)

    for(i in 1:n){
        grade[i]<-as.character(Tu[[i]]$Grade[1])
        mort[i]<-Tu[[i]]$Mortality[1]
        ID[i]<-Tu[[i]][1,1]
    }

    resultDF <- data.frame(ID = ID,grade = grade, mort = mort)

    ## set up results columns
    resultDF$thomas.T.D <- resultDF$thomas.T.lam <- resultDF$thomas.T.sig <- rep(NA,n)
    resultDF$thomas.S.D <- resultDF$thomas.S.lam <- resultDF$thomas.S.sig <- rep(NA,n)
    resultDF$matern.T.D <- resultDF$matern.T.lam <- resultDF$matern.T.tau <- rep(NA,n)
    resultDF$matern.S.D <- resultDF$matern.S.lam <- resultDF$matern.S.tau <- rep(NA,n)
    resultDF$void.T.Dp <- resultDF$void.T.Dc <- resultDF$void.T.tau <- rep(NA,n)
    resultDF$void.S.Dp <- resultDF$void.S.Dc <- resultDF$void.S.tau <- rep(NA,n)
     
    ## write patient info out to a file
    ## write.csv(resultDF,file = "resultsDF.csv",row.names = FALSE)

    ## change number of cores used
    ## create a cluster with ncores cores
    cl <- makeSOCKcluster(ncores)
    registerDoSNOW(cl)
    progress <-  function(x) cat(sprintf("T Thomas model %d \n", x))
    opts <- list(progress=progress)

Fit Thomas process
------------------

    ### Tumour

    ### Tumour

    resultDF[,c(6,5,4)] <- foreach(i = 1:n, 
                  .combine = "rbind", 
                  .packages = "palm",
                  .options.snow=opts) %dopar% {
        fit <- fit.ns(points=points[[i]],lims=rbind(c(0,1),c(0,1)),R = R,
                      start = c(sigma = start[i,4], lambda = start[i,5], D = start[i,6]))
        coef(fit)[1:3]
    }
    ### Stoma
    progress <-  function(x) cat(sprintf("S Thomas model %d \n", x))
    opts <- list(progress=progress)

    resultDF[,c(9,8,7)] <- foreach(i = 1:n, 
                  .combine = "rbind", 
                  .packages = "palm",
                  .options.snow=opts) %dopar% {
        fit <- fit.ns(points=pointss[[i]],lims=rbind(c(0,1),c(0,1)),R = R,
                      start = c(sigma = start[i,7], lambda = start[i,8], D = start[i,9]))
        coef(fit)[1:3]
    }
    ## write patient info out to a file
    ## write.csv(resultDF,file = "resultsDF.csv",row.names = FALSE)

Fit Matérn process
------------------

    ### Tumour
    progress <-  function(x) cat(sprintf("T Matern model %d \n", x))
    opts <- list(progress=progress)

    resultDF[,c(12,11,10)] <- foreach(i = 1:n, 
                  .combine = "rbind", 
                  .packages = "palm",
                  .options.snow=opts) %dopar% {
        fit <- fit.ns(points=points[[i]],lims=rbind(c(0,1),c(0,1)),R = R,disp = "uniform",
                      start = c(tau = start[i,10], lambda = start[i,11], D = start[i,12]))
        coef(fit)[1:3]
    }


    ### Stoma

    progress <-  function(x) cat(sprintf("S Matern model %d \n", x))
    opts <- list(progress=progress)

    resultDF[,c(15,14,13)] <- foreach(i = 1:n, 
                  .combine = "rbind", 
                  .packages = "palm",
                  .options.snow=opts) %dopar% {
                      fit <- fit.ns(points=pointss[[i]],lims=rbind(c(0,1),c(0,1)),R = R,disp = "uniform",
                                    start = c(tau = start[i,13], lambda = start[i,14], D = start[i,15]))
        coef(fit)[1:3]
    }
    ## write patient info out to a file
    ## write.csv(resultDF,file = "resultsDF.csv",row.names = FALSE)

Fit Void process
----------------

    ### Tumour

    progress <-  function(x) cat(sprintf("T void model %d \n", x))
    opts <- list(progress=progress)

    resultDF[,c(18,17,16)] <- foreach(i = 1:n, 
                  .combine = "rbind", 
                  .packages = "palm",
                  .options.snow=opts) %dopar% {
                      fit <- fit.void(points=points[[i]],lims=rbind(c(0,1),c(0,1)),R = R,
                                      bounds = list(Dp = c(0,70)),
                                      start = c(tau = start[i,16], Dc = start[i,17], Dp = start[i,18]))
        coef(fit)[1:3]
    }


    ### Stoma

    progress <-  function(x) cat(sprintf("S void model %d \n", x))
    opts <- list(progress=progress)

    resultDF[,c(21,20,19)] <- foreach(i = 1:n, 
                  .combine = "rbind", 
                  .packages = "palm",
                  .options.snow=opts) %dopar% {
                      fit <- fit.void(points=pointss[[i]],lims=rbind(c(0,1),c(0,1)),R = R,
                                      bounds = list(Dp = c(0,70)),
                                      start = c(tau = start[i,19], Dc = start[i,20], Dp = start[i,21]))
        coef(fit)[1:3]
    }
    ## write patient info out to a file
    ## write.csv(resultDF,file = "resultsDF.csv",row.names = FALSE)

Bootstrap functions
-------------------

    ## read in csv of parameters (as above)
    df <- read.csv("resultsDF.csv")


    ############# Heirarchical bootstrap
    resample <- function(dat, cluster, replace) {
      # sample the clustering factor
      cls <- sample(unique(dat[[cluster[1]]]), replace=replace[1])
      # subset on the sampled clustering factors
      sub <- lapply(cls, function(b) subset(dat, dat[[cluster[1]]]==b))
      # join and return samples
      do.call(rbind, sub)

    }

    ## Function to do a heirarchical bootstrap for CRC data.
    ## This function resamples based on a nested level (i.e., as each patient has multiple slides)
    ## Arguments are, 1) data a named list of numeric vectors to be bootstrapped 2) nsim times based on the factor
    ## vector 3) patient
    ## By default the "median" of the bootstrapped resamples (again median resamples) for each list element (data) are returned
    ## along with 95% quantiles

    bootstrap <- function(data, nsim, patient, value = "median"){
        pb <- txtProgressBar(style = 3)
        cluster = "patient"
        boots <- list()
        for(i in 1:length(data)){
            dat <- data.frame(measurement = data[[i]],patient = patient)
            boots[[i]] <- replicate(nsim, median(resample(dat, cluster, TRUE)$measurement,na.rm = TRUE))
            setTxtProgressBar(pb,i)
        }
        close(pb)
        val <- sapply(boots,value)
        quantiles <- sapply(boots,quantile,c(0.025,0.975))
        res <- rbind(val = val, quantiles)
        colnames(res) <- names(data)
        return(res)
    }

    ## Function to chose the data to split, and the level at which to split to perform the heirarchical bootstrap above.
    ## name should be a character vector of the data one wishes to bootstrap.
    ## by a single character of the level at which to split (i.e., mortality of patient).
    ## ID a single character specifying the name of the column in df whuch contains
    ## the patient IDs.
    ## df a matrix of data with columns named name, by and ID as above.
    ## nsim an integer specifying the number of bootstraps to do.
    ## the function returns a nested list of the output of the bootstrap function (above)
    ## for each chosen level.

    fun <- function(name, by, ID, df, nsim){
        d <- list()
        for (i in 1:length(name)){
            d[[i]] <-  split(df[[name[i]]],df[[by]])
        }
        pat <- split(df[[ID]],df[[by]])
        names(d) <- name
        boots <- list()
        for (i in 1:length(d)){
            boots[[i]] <- list()
            for(j in 1:length(d[[i]])){
                boots[[i]][[j]] <- bootstrap(data = d[[i]][j],nsim = nsim, patient = pat[[j]])
            }
        }
        names(boots) <- name
        boots
    }

For example to bootstrap the void process dispersion parameters for both
the Tumour and Stroma nuclei, the following code should be run.

    res <- fun(c("void.T.tau","void.S.tau"),"grade","ID", df,1000)

References
==========

Jones-Todd, C. M, P Caie, J Illian, B. C Stevenson, Savage A, D
Harrison, and J Bown. 2017. “Identifying Unusual Structures Inherent in
Point Pattern Data and Its Application in Predicting Cancer Patient
Survival.” *ArXiv Preprint ArXiv:1705.05938*.

Stevenson, Ben C. 2017. *Palm: Fitting Point Process Models Using the
Palm Likelihood*. <https://github.com/b-steve/palm>.
