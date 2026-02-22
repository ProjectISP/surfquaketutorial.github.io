# Explore Data Avalability

I personally think that it really worht it explore your data avalability before any further analysis. It is very interesing to have a tool that can show you the data availability inside your folder tree. For this reason surfquake offers a command line tool that easily will show you a plot with lines where you have data, no data and gaps in red.

For big data, it can last a little bit but it shows you the progress on screen. 

## Command Line

```bash
surfquake explore -w [Path to your mseed files, e.g., './data/*']
```

## Example

In the example, I simply navigate to the forder where I have placed three stations with three channels so nine files, then I run the command:

```bash
surfquake explore -w "./*"
```

![Screenshot](img/explore.png)
