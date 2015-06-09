# templates for Flocker examples

The `flocker-tutorials` folder contains 2 sub-folders, each one contains the files for a tutorial using Flocker:

 * [tutorial-1](http://build.clusterhq.com/results/docs/newdoc-tryflocker-FLOC-1920/build-7423/using/tutorial/tryflocker.html#step-3-deploying-an-app-on-the-first-host)
 * [tutorial-2](http://build.clusterhq.com/results/docs/newdoc-tryflocker-FLOC-1920/build-7423/using/tutorial/index.html)

The deployment files are templated using Mustache and expect 2 variables to be populated:

 * node1_public_ip
 * node2_public_ip

The flocker-tutorials folder should be copied out to: '/flocker-tutorials'

NOTE - for the mongodb tutorial - the user will need a Mongo client and this will need installing.

`sudo apt-get install mongodb-clients` is how the tutorial says to install it.