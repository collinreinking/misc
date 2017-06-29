<SPAN STYLE="font-size: 75%">Much of what follows is adapted from<a href="https://chrisalbon.com/jupyter/run_project_jupyter_on_amazon_ec2.html"> Run Jupyter on amazon ec2</a> starting at <b>Create a password for jupyter notebook</b>.(don't install anaconda) with modifications figured out by <a href="https://www.linkedin.com/in/james-nguyen-6575a431/"> James Nguyen</a>.  Any errors were likely introduce by me, <a href="https://www.linkedin.com/in/collin-reinking/">Collin Reinking</a></SPAN>.  

I recommend simply copying and pasting the code chunks where possible (as opposed to re-typing them by hand).

## Getting iPython notebooks up and running on your UCB AMI of Ec2

### Adjust security group settings
Log in to the aws console https://console.aws.amazon.com/ec2  
Select `Security Groups` from the left hand side  
Click the box next to the security group you are using (probably still called `Hadoop Cluster UCB`)  
Click the tab for `Inbound`  
Click the `Edit` button  
Click `Add Rule` (at bottom of pop up window)  
Change the new rule to HTTPS  
Click `Add Rule` again  
Don't the new rule to type(`Custom TCP Rule`)  
Change the port to `8888`  
Change availability to from `Custom` to `Anywhere`  
Click Save  

### Log into your ec2 instance
Do this however you do it.  If you haven't set up an elastic IP to make this easier for yourself... you should.

### Install ipython notebook on your Instance
**Execute**
```bash
pip install ipython notebook
```

### Create a password for jupyter notebook
In order to get into ipython **execute**
```bash
ipython
```

Now that you're in iPython your command prompt should look a little different(colors and whatnot).

In iPython **execute**
```python
from IPython.lib import passwd
my_pass = passwd()
```
you will be prompted for a password.  To be clear, this is you *choosing* the password that you will use to access the jupyter notebook running on your ec2. You will enter this twice then the function will return an encrypted password that will be stored into `my_pass`(to be used later).  Check to make sure that there is a value there by **executing** `print my_pass`, it should look something like this:

> sha1:98ff0e580111:12798c72623a6eecd54b51c006b1050f0ac1a62d

We are going to need this value later but we're going to save it in a bit of an odd way so that we don't have to worry about where it needs to be put.    
**Execute** the following commands (still in python):
```python
text_file = open("hashed_my_pass.txt", "w")
text_file.write("export NOTEBOOK_PASS=%s" % my_pass)
text_file.close()
exit
```
You should be back on the in your bash shell on your ec2.  
Now **Execute**
```bash
$(head -n 1 hashed_my_pass.txt)
```
This will execute the line of text in the `hashed_my_pass.txt` as a bash command which will create an environment variable called `NOTEBOOK_PASS` that has the hashed password that you had stored in `my_pass` back in python.  Note:  we shouldn't need `hashed_my_pass.txt` anymore so you can `rm hashed_my_pass.txt` if you'd like to.

### Create certificates for https
```bash
mkdir /home/w205/certs
cd /home/w205/certs
sudo openssl req -x509 -nodes -days 365 -newkey rsa:1024 -keyout mycert.pem -out mycert.pem
cd ~
```
You will be asked some questions, just go ahead and answer them.  the final result is a file called `jupiter_notebook_cert.pem`.

### Configuring jupyter
There is a block of code that we need to add to the start of a file called `ipython_notebook_config.py` inside of `~/.ipython/profile_pyspark` but the code below will do this for you.

**COPY AND PASTE** the following block of code directly into terminal.  It will automatically pull in the encrypted password that we got from ipython earlier.

```bash
###start of block to copy and paste into ec2 terminal###

#create a txt file with the code that you need to insert into the config file.
cat > config_block.txt <<EOF
c = get_config()

# Kernel config
c.IPKernelApp.pylab = 'inline'  # if you want plotting support always in your notebook

# Notebook config
c.NotebookApp.certfile = u'/home/w205/certs/mycert.pem' #location of your certificate file
c.NotebookApp.ip = '*'
c.NotebookApp.open_browser = False  #so that the ipython notebook does not opens up a browser by default
c.NotebookApp.password = u'$NOTEBOOK_PASS'  #the encrypted password we generated earlier
# It is a good idea to put it on a known, fixed port
c.NotebookApp.port = 8888

EOF
#create a new file that has the new code block and then the original config file.
cat config_block.txt ~/.ipython/profile_pyspark/ipython_notebook_config.py > ~/.ipython/profile_pyspark/ipython_notebook_config_temp.py
#remove the original config file
rm ~/.ipython/profile_pyspark/ipython_notebook_config.py
#rename the new file so that it has the name of the config file.
mv ~/.ipython/profile_pyspark/ipython_notebook_config_temp.py ~/.ipython/profile_pyspark/ipython_notebook_config.py
#make sure w205 can also use the updated profile
cp -rf ~/.ipython/profile_pyspark /home/w205/.ipython/



###END of block to copy and paste into ec2 terminal###
```

You should be ready to go now.

You can get ipython notebooks running by executing:
```bash
ipython notebook --profile pyspark
```

Then open a browser window and go to your `https://<instance url>:8888`  
You may be told that you connection isn't private.  
just click where it says `ADVANCED` and then click the link at the bottom that says "Proceed To..."  
You will be asked to type in a password, this password should be the one that you entered in python near the start of this walkthrough.  
