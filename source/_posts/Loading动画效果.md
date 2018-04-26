---
title: "Loading 动画效果"
date: "2018-04-13"
categories: 
    - "animation"
---
# Loading动画效果

项目地址：[Github](https://github.com/81813780/AVLoadingIndicatorView)

动画效果：

![loading动画](https://github.com/81813780/AVLoadingIndicatorView/blob/master/screenshots/avi.gif)

进阶升级，主要是一些自定义的view，基本原理是在draw的时候，因某些属性的变化，使得draw的图形发生变化，从而产生动画效果

## 基础类

```
public abstract class Indicator extends Drawable implements Animatable
```

可以详细了解下Drawable的一些常用方法

## BallPulseIndicator

三个小圆点，大小相继变化

```
/**
 * Created by Jack on 2015/10/16.
 */
public class BallPulseIndicator extends Indicator {

    public static final float SCALE=1.0f;

    //scale x ,y
    private float[] scaleFloats=new float[]{SCALE,
            SCALE,
            SCALE};



    @Override
    public void draw(Canvas canvas, Paint paint) {
        float circleSpacing=4;
        float radius=(Math.min(getWidth(),getHeight())-circleSpacing*2)/6;
        //根据view的宽度高度，和圆圈的间隔大小计算每个元的半径
        
        float x = getWidth()/ 2-(radius*2+circleSpacing);
        float y=getHeight() / 2;
        第一个小圆圈的位置，
        for (int i = 0; i < 3; i++) {//总共有三个，每个小圆圈的位置相继向右偏移
            canvas.save();
            float translateX=x+(radius*2)*i+circleSpacing*i;
            
            canvas.translate(translateX, y);
           // Log.i("BallPulseIndicator","scaleFloats["+i+"]="+scaleFloats[i]);
            canvas.scale(scaleFloats[i], scaleFloats[i]);//scaleFloats[i]根据ValueAnimator的值来变化
            canvas.drawCircle(0, 0, radius, paint);
            canvas.restore();
        }
    }

    @Override
    public ArrayList<ValueAnimator> onCreateAnimators() {
        ArrayList<ValueAnimator> animators=new ArrayList<>();
        int[] delays=new int[]{120,240,360};
        for (int i = 0; i < 3; i++) {
            final int index=i;

            ValueAnimator scaleAnim=ValueAnimator.ofFloat(1,0.3f,1);

            scaleAnim.setDuration(750);
            scaleAnim.setRepeatCount(-1);
            scaleAnim.setStartDelay(delays[i]);//每个小圈的scale变化通过delay来错开

            addUpdateListener(scaleAnim,new ValueAnimator.AnimatorUpdateListener() {
                @Override
                public void onAnimationUpdate(ValueAnimator animation) {
                    scaleFloats[index] = (float) animation.getAnimatedValue();
                    postInvalidate();
                }
            });
            animators.add(scaleAnim);
        }
        return animators;
    }
}
```

通过代码分析，基本原理，有一种豁然开朗的感觉，有时候觉得动画这东西就是在变魔术



## BallClipRotateIndicator



```
* Created by Jack on 2015/10/16.
 */
public class BallClipRotateIndicator extends Indicator {

    float scaleFloat=1,degrees;

    @Override
    public void draw(Canvas canvas, Paint paint) {
        paint.setStyle(Paint.Style.STROKE);
        paint.setStrokeWidth(3);

        float circleSpacing=12;
        float x = (getWidth()) / 2;
        float y=(getHeight()) / 2;
        canvas.translate(x, y);
        canvas.scale(scaleFloat, scaleFloat);
        canvas.rotate(degrees);
        RectF rectF=new RectF(-x+circleSpacing,-y+circleSpacing,0+x-circleSpacing,0+y-circleSpacing);
        canvas.drawArc(rectF, -45, 270, false, paint);//画有一段弧形
    }

    @Override
    public ArrayList<ValueAnimator> onCreateAnimators() {
        ArrayList<ValueAnimator> animators=new ArrayList<>();
        ValueAnimator scaleAnim=ValueAnimator.ofFloat(1,0.6f,0.5f,1);
        scaleAnim.setDuration(750);
        scaleAnim.setRepeatCount(-1);
        addUpdateListener(scaleAnim,new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                scaleFloat = (float) animation.getAnimatedValue();
                postInvalidate();
            }
        });
        ValueAnimator rotateAnim=ValueAnimator.ofFloat(0,180,360);
        rotateAnim.setDuration(750);
        rotateAnim.setRepeatCount(-1);
        addUpdateListener(rotateAnim,new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                degrees = (float) animation.getAnimatedValue();
                postInvalidate();
            }
        });
        animators.add(scaleAnim);
        animators.add(rotateAnim);
        return animators;
    }
}
```



## SquareSpinIndicator

上下左右翻转的正方形，通过camera的api来用于计算3d的转化，用于生成matrix提供给canvas使用

```

/**
 * Created by Jack on 2015/10/16.
 */
public class SquareSpinIndicator extends Indicator {

    private float rotateX;
    private float rotateY;

    private Camera mCamera;
    private Matrix mMatrix;

    public SquareSpinIndicator(){
        mCamera=new Camera();
        mMatrix=new Matrix();
    }

    @Override
    public void draw(Canvas canvas, Paint paint) {

        mMatrix.reset();
        mCamera.save();
        mCamera.rotateX(rotateX);
        mCamera.rotateY(rotateY);//通过camera来坐x轴y轴的旋转
        mCamera.getMatrix(mMatrix);//生成matrix
        mCamera.restore();

        mMatrix.preTranslate(-centerX(), -centerY());
        mMatrix.postTranslate(centerX(), centerY());
        //以上两行主要解决中心对称问题
        //转变之前将0.0的中心点转化到view的中心，转化之后，再恢复回去
        canvas.concat(mMatrix);

        canvas.drawRect(new RectF(getWidth()/5,getHeight()/5,getWidth()*4/5,getHeight()*4/5),paint);
    }

    @Override
    public ArrayList<ValueAnimator> onCreateAnimators() {
        ArrayList<ValueAnimator> animators=new ArrayList<>();
        ValueAnimator animator=ValueAnimator.ofFloat(0,180,180,0,0);
        addUpdateListener(animator,new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                rotateX= (float) animation.getAnimatedValue();
                postInvalidate();
            }
        });
        animator.setInterpolator(new LinearInterpolator());
        animator.setRepeatCount(-1);
        animator.setDuration(2500);

        ValueAnimator animator1=ValueAnimator.ofFloat(0,0,180,180,0);
        addUpdateListener(animator1,new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                rotateY= (float) animation.getAnimatedValue();
                postInvalidate();
            }
        });
        animator1.setInterpolator(new LinearInterpolator());
        animator1.setRepeatCount(-1);
        animator1.setDuration(2500);

        animators.add(animator);
        animators.add(animator1);
        return animators;
    }

}

```



