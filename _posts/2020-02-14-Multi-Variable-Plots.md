# Encoding Variables into RGB for 5D Representations

> Building 5D Visualizations within 2D by encoding 3 parameters within the RGB color values. This is especially useful for visualizing filter or fequency values. It is also useful for classification certainty or other value representations, especially those where the interest lies in the relative strengths.

## Setup


```python
import numpy as np
import matplotlib.pyplot as plt
from pprint import pprint
import os
```

## Utility Functions


```python
def save(fig, name, savepath=savepath):
    if not os.path.exists(savepath+'svg/'):
        os.makedirs(savepath+'svg/')
    fig.savefig(savepath+'svg/{}.svg'.format(name))
    fig.savefig(savepath+'{}.jpg'.format(name))
    return True
```

## Loading Data


```python
savepath = 'img/'
filename = 'data/logT_4_logA_8.5_mag_24.5.npy'
```


```python
data = np.load(filename)
```


```python
x = data['RA']
y = data['DEC']
```

## Astronomical adjustements


```python
x = (np.max(x)-x)*np.cos(np.pi/180*y)*3600
y = (y-np.min(y))*3600
```


```python
from astropy import coordinates
from astropy import units
```


```python
c = coordinates.SkyCoord(ra=data['RA']*units.degree, dec=data['DEC']*units.degree, distance=770*units.kpc)
(data['RA'] == np.array(c.ra)).all(), (data['DEC'] == np.array(c.dec)).all()
```




    (True, True)



# Three Color Plot

## General Functions


```python
from matplotlib.path import Path
from scipy import ndimage
```


```python
def setup_fig(shape=(1,1), figsize=(12,12), xlabel='RA', ylabel='DEC', invert_xaxis=True):
    fig, axarr = plt.subplots(*shape, figsize=figsize)
    if shape[0]>1 or shape[1]>1:
        for ax in axarr:
            ax.set_xlabel(xlabel)
            ax.set_ylabel(ylabel)
            if invert_xaxis:
                ax.invert_xaxis()
    fig.tight_layout()
    return fig, axarr
```

## Scaling Functions


```python
def rescale(arr):
    ''' rescale array to 0-1 '''
    return (arr-arr.min())/(arr.max()-arr.min())
```


```python
def rescale_hex(arr):
    ''' rescale array to 0-255 '''
    return np.round((arr-arr.min())*255/(arr.max()-arr.min())).astype(np.int)
```


```python
def get_color(val1, val2, val3):
    ''' convert RGB values 0-255 to hexcode '''
    #vhex = np.vectorize(hex)
    return '#{:02X}{:02X}{:02X}'.format(val1, val2, val3)

vget_color = np.vectorize(get_color)  # vectorize function
```


```python
def three_var_plot(ax, x, y, arr1, arr2, arr3, s=1.5):
    ''' convert three arrays into a normalized color value '''
    rel1 = np.round(rescale_hex(arr1)).astype(np.int)
    rel2 = np.round(rescale_hex(arr2)).astype(np.int)
    rel3 = np.round(rescale_hex(arr3)).astype(np.int)
    colors = vget_color(rel1, rel2, rel3)
    alphas = np.mean(np.array([rel1, rel2, rel3]), axis=0)/255
    idx_order = np.argsort(alphas)
    # Set color
    ax.set_facecolor((0, 0, 0))
    ax.scatter(x[idx_order], y[idx_order], color=colors[idx_order], s=s)
```

## Testing various scalings


```python
fig, axarr = plt.subplots(3,1,figsize=(10,15))
three_var_plot(axarr[0], data['RA'], data['DEC'], 
               data['HST_WFC3_F110W'], data['HST_WFC3_F160W'], data['HST_WFC3_F275W'])
axarr[0].invert_xaxis()
# explicit
three_var_plot(axarr[1], x, y, 
               data['HST_WFC3_F110W'], data['HST_WFC3_F160W'], data['HST_WFC3_F275W'])
# astropy
c = coordinates.SkyCoord(ra=data['RA']*units.degree, dec=data['DEC']*units.degree, distance=770*units.kpc).cartesian
three_var_plot(axarr[2], c.x, c.y, 
               data['HST_WFC3_F110W'], data['HST_WFC3_F160W'], data['HST_WFC3_F275W'])
```


![png]({{ site.baseurl }}/assets/2020-02-14-Multi-Variable-Plots/output_24_0.png)


## Create Color Triangle


```python
def make_equilateral(vertex1, vertex2, positive=True):
    """ create vertices of the triangle from a position in the 2D space of the plot """
    # Retype to array
    if type(vertex1) == tuple:
        vertex1 = np.array(vertex1)
    if type(vertex2) == tuple:
        vertex2 = np.array(vertex2)
    # Adjust difference
    diff = vertex1-vertex2
    if min(diff) < 0:
        vertex1, vertex2 = vertex2, vertex1
        diff *= -1
    if diff[0] == 0 and diff[1] != 0:
        # equal x
        a = diff[1]
        h = a*np.sqrt(3)/2
        if positive:
            vertex3 = [vertex1[0]+h, min(vertex1[1], vertex2[1]) + diff[1]/2]
        else:
            vertex3 = [vertex1[0]+h, min(vertex1[1], vertex2[1]) - diff[1]/2]
    elif diff[1] == 0 and diff[0] != 0:
        # equal y
        a = diff[0]
        h = a*np.sqrt(3)/2
        if positive:
            vertex3 = [min(vertex1[0], vertex2[0]) + diff[0]/2, vertex1[1]+h]
        else:
            vertex3 = [min(vertex1[0], vertex2[0]) + diff[0]/2, vertex1[1]-h]
    else:
        raise
    # Return vertices of triangle
    return [vertex1, vertex2, vertex3]
```


```python
def add_triangle_colorbar(ax, vertices, labels, 
                          xres=200, yres=200, text_color='k', upright=False, 
                          legend_alpha=0.8, smooth_size=3):
    """ add triangular equilateral triangle as the 'colorbar' """
    # Triangular Colorbar
    if len(vertices) == 2:
        triangle_vertices = np.array(make_equilateral(*vertices, positive=upright))
    else:
        triangle_vertices = vertices
    # Ensure vertices extracted properly
    xmin, ymin = triangle_vertices.min(axis=0)
    xmax, ymax = triangle_vertices.max(axis=0)
    # Create 2D mesh
    x, y = np.meshgrid(np.linspace(xmin, xmax, xres), np.linspace(ymin, ymax, yres))
    xmax = x.max()
    xmin = x.min()
    ymax = y.max()
    ymin = y.min()
    # RGB Colors Fraction, allow upside down triabgle too
    top = (y-ymin)/(ymax-ymin)
    if not upright:
        top = 1 - top
    left  = (x-xmin)/(xmax-xmin)
    right = 1-left
    # Create Path, Mask of Triangle, and Alpha 
    triangle_path = Path(triangle_vertices)
    coords = np.hstack((x.reshape(-1, 1), y.reshape(-1, 1)))
    mask = triangle_path.contains_points(coords).reshape(yres, xres).astype(float)
    alpha = ndimage.generic_filter(mask, np.nanmean, 
                                   size=smooth_size, mode='constant', cval=0)*legend_alpha
    # Combine for colors to create the shading
    colors = np.dstack((top*1.5, left, right))
    colors = colors/colors.max(axis=2)[:,:,None]
    colors = np.dstack((colors, alpha))
    # Plot triangle as image
    ax.imshow(colors, extent=(xmin, xmax, ymin, ymax), 
              origin='lower', interpolation='bilinear')
    # Annotate with labels
    triangle_vertices = np.roll(triangle_vertices, 1, axis=0)
    for i, txt in enumerate(labels):
        xcoord = triangle_vertices[i][0]
        ycoord = triangle_vertices[i][1]
        if ycoord == ymax:
            va = 'bottom'
            xytext = (0, 3)
        elif ycoord == ymin:
            va = 'top'
            xytext = (0, -3)
        ax.annotate(txt, (xcoord, ycoord), color=text_color, ha='center', va=va,
                   textcoords='offset pixels', xytext=xytext)
    
    return
```

## Show Triangular representation


```python
fig, axarr = setup_fig(shape=(3,1), figsize=(10,20))
color_list = [['HST_WFC3_F110W', 'HST_WFC3_F160W', 'HST_WFC3_F275W'],
              ['HST_WFC3_F275W', 'HST_WFC3_F336W', 'HST_ACS_WFC_F814W'],
              ['HST_WFC3_F110W', 'HST_WFC3_F275W', 'HST_ACS_WFC_F814W']]
for i, colors in enumerate(color_list):
    add_triangle_colorbar(axarr[i], vertices=[(10.7, 42.25), (10.9, 42.25)], 
                          labels=[color.split('_')[-1] for color in colors],
                          text_color='#DDDDDD')
    three_var_plot(axarr[i], data['RA'], data['DEC'], 
                   data[colors[0]], data[colors[1]], data[colors[2]])
    axarr[i].invert_xaxis()
    axarr[i].set_xlabel('RA')
    axarr[i].set_ylabel('DEC')
```


![png]({{ site.baseurl }}/assets/2020-02-14-Multi-Variable-Plots/output_29_0.png)



