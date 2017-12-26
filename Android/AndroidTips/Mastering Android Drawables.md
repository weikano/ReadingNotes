## What is Drawable
> An abstraction of an entity that can be drawn on to a Canvas. Contrary to View a Drawable doesnt deal with measure/layout it only draw Canvas. 

## How to get Drawable object

```
context.getResource().getDrawable(int id);
Drawable.createFrom();
```

    
![image](https://raw.githubusercontent.com/weikano/NoteResources/master/583d74f2ab6441378101c1ea.png)

**getDrawable(int id)** always return a new Drawable instance, but **Drawables share their ConstantStates when decode from the same resource**

**calling Drawable.mutate copies the ConstantStates**

## Xml tag versus Drawable
![image](https://raw.githubusercontent.com/weikano/NoteResources/master/583d74f2ab6441378101c1e8.png)

## Some tips for <bitmap> tag

```
<bitmap 
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:src="@drawable/xxxx"
        android:gravity="center" // if gravity is not set then the src will be stretched to fit the whole View space
        android:tileMode="repeat|mirror" // could do more then simple bitmap
        />
```

## **ColorDrawable is not clipped prior to HoneyComb**
![image](https://raw.githubusercontent.com/weikano/NoteResources/master/583d74f2ab6441378101c1e9.png)

## Advance Drawable Usage
- Using AnimateDrawable in Custom View
>be aware of Drawable.Callback interface, just override View.verifyDrawable(Drawable)

- Flattenning View hierarchy
> Using android:windowBackground Drawable can minimized layout&better launching animation

## **Conclusion**
- Drawable are **light-weighted**
- Drawable are **system-cached**

> Like background drawbale of normal button, the system cached the drawable so it will not occupy your memory

- Drawable are **config-dependent**
- Drawable are **easy-to-use**