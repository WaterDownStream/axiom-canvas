# Representative source excerpts

These excerpts are copied from the supplied source snapshot. Full files, dependencies and tests are in the source ZIP.

## app/src/main/java/io/axiom/canvas/Measurements.java

```java
package io.axiom.canvas;

import android.graphics.*;
import java.util.*;
import static io.axiom.canvas.GeometryModel.*;

/** One measurement source shared by canvas, inspector and vector export. */
final class Measurements {
    private Measurements() {}
    static boolean supported(GeoObject o){return !(o instanceof GraphObject||o instanceof TextObject||o instanceof FreehandObject);}
    static String[] values(GeoObject o,Viewport v){
        if(o instanceof TwoPointObject){TwoPointObject l=(TwoPointObject)o;return new String[]{(l.kind==Kind.LINE||l.kind==Kind.RAY?"Anchor span  ":"Length  ")+lengthText(distance(l.a,l.b),v)};}
        if(o instanceof PointObject){PointF p=((PointObject)o).point;return new String[]{"x  "+coordinateText(p.x,v),"y  "+coordinateText(p.y,v)};}
        if(o instanceof CircleObject){double r=((CircleObject)o).radius();return new String[]{"Radius  "+lengthText(r,v),"Diameter  "+lengthText(2*r,v),"Circumference  "+lengthText(2*Math.PI*r,v),"Area  "+areaText(Math.PI*r*r,v)};}
        if(o instanceof EllipseObject){EllipseObject e=(EllipseObject)o;return new String[]{"Radii  "+lengthText(e.radiusX(),v)+" / "+lengthText(e.radiusY(),v),"Area  "+areaText(Math.PI*e.radiusX()*e.radiusY(),v)};}
        if(o instanceof ArcObject){ArcObject a=(ArcObject)o;double sweep=normalizeSweep(screenAngle(a.center,a.end)-screenAngle(a.center,a.start));return new String[]{"Sweep  "+angleText(sweep,v),"Arc length  "+lengthText(distance(a.center,a.start)*Math.toRadians(sweep),v)};}
        if(o instanceof PolyObject){PolyObject p=(PolyObject)o;if(!p.closed()){double a=angleDegrees(p.vertices.get(0),p.vertices.get(1),p.vertices.get(2));return new String[]{"Angle  "+(Double.isFinite(a)?angleText(a,v):"undefined")};}return new String[]{"Area  "+areaText(polygonArea(p.vertices),v),"Perimeter  "+lengthText(polygonPerimeter(p.vertices),v)};}
        return new String[0];
    }
    static PointF anchor(GeoObject o,Viewport v){
        PointF p=v.toScreen(o.center());
        if(o instanceof CircleObject){CircleObject c=(CircleObject)o;p=v.toScreen(c.center);p.y-=c.radius()*v.scale;}
        else if(o instanceof EllipseObject){EllipseObject e=(EllipseObject)o;p=v.toScreen(e.center);p.y-=e.radiusY()*v.scale;}
        else if(o instanceof PointObject)p=v.toScreen(((PointObject)o).point);
        p.x+=o.labelDx*v.scale;p.y-=o.labelDy*v.scale;
        return p;
    }
    static boolean shown(GeoObject o,boolean interactive){return supported(o)&&(o.measured||(interactive&&o.selected));}
    static void draw(Canvas c,GeoObject o,Viewport v,float density){
        String[] rows=values(o,v);if(rows.length==0)return;
        // Circle has a compact canvas summary; the inspector contains all four values.
        if(o instanceof CircleObject)rows=new String[]{rows[0],rows[3]};
        Paint p=new Paint(Paint.ANTI_ALIAS_FLAG);p.setTypeface(Typeface.create("sans-serif-medium",Typeface.NORMAL));p.setTextSize(12*density);
        float maxWidth=Math.max(90*density,v.width-24*density),width=0;
        for(String row:rows)width=Math.max(width,p.measureText(row));width=Math.min(maxWidth,width+26*density);
        float h=(rows.length*19+16)*density;PointF a=anchor(o,v);
        float x=Math.max(12*density,Math.min(v.width-width-12*density,a.x-width/2));
        float y=Math.max(12*density,Math.min(v.height-h-66*density,a.y-h-20*density));
        o.labelBounds.set(x,y,x+width,y+h);
        p.setColor(v.darkCanvas?0xFA242B3B:0xFAFFFFFF);c.drawRoundRect(o.labelBounds,10*density,10*density,p);
        p.setColor(o.color);c.drawRoundRect(new RectF(x,y,x+3*density,y+h),2*density,2*density,p);
        p.setColor(v.darkCanvas?0xFFF3F4FF:0xFF292C3D);
        for(int i=0;i<rows.length;i++){String row=rows[i];float available=width-24*density;if(p.measureText(row)>available)p.setTextSize(Math.max(8*density,12*density*available/p.measureText(row)));c.drawText(row,x+12*density,y+(21+19*i)*density,p);p.setTextSize(12*density);}
    }
    static String report(GeoObject o,Viewport v){return String.join("\n",values(o,v));}
}
```

## app/src/main/java/io/axiom/canvas/ExpressionParser.java

```java
package io.axiom.canvas;

import java.util.Locale;

/** Offline expression compiler shared by graphing and table formulas. */
public final class ExpressionParser {
    public interface Expression { double evaluate(double x); }
    public interface Variables { double value(String name); }
    public interface VariableExpression { double evaluate(Variables variables); }
    public static final class ParseException extends IllegalArgumentException { private static final long serialVersionUID=1L; ParseException(String message){super(message);} }
    private interface Node { double eval(Variables variables); }
    private final String source; private int position;
    private ExpressionParser(String value){source=value==null?"":value.toLowerCase(Locale.US).replace("π","pi");}
    public static Expression compile(String value){VariableExpression e=compileVariables(value);return x->e.evaluate(n->n.equals("x")||n.equals("t")||n.equals("theta")?x:0);}
    public static VariableExpression compileVariables(String value){ExpressionParser p=new ExpressionParser(value);Node root=p.conditional();p.spaces();if(p.position!=p.source.length())throw p.error("Unexpected '"+p.source.charAt(p.position)+"'");return variables->{try{double result=root.eval(variables);return Double.isFinite(result)?result:Double.NaN;}catch(ArithmeticException ignored){return Double.NaN;}};}
    private Node conditional(){Node condition=comparison();if(eat('?')){Node yes=conditional();expect(':');Node no=conditional();return v->truth(condition.eval(v))?yes.eval(v):no.eval(v);}return condition;}
    private Node comparison(){Node left=expression();while(true){String op=comparisonOperator();if(op==null)return left;Node a=left,b=expression();switch(op){case"<":left=v->a.eval(v)<b.eval(v)?1:0;break;case"<=":left=v->a.eval(v)<=b.eval(v)?1:0;break;case">":left=v->a.eval(v)>b.eval(v)?1:0;break;case">=":left=v->a.eval(v)>=b.eval(v)?1:0;break;case"==":left=v->Math.abs(a.eval(v)-b.eval(v))<1e-9?1:0;break;default:left=v->Math.abs(a.eval(v)-b.eval(v))>=1e-9?1:0;}}
    }
    private Node expression(){Node left=term();while(true){if(eat('+')){Node a=left,b=term();left=v->a.eval(v)+b.eval(v);}else if(eat('-')){Node a=left,b=term();left=v->a.eval(v)-b.eval(v);}else return left;}}
    private Node term(){Node left=unary();while(true){if(eat('*')){Node a=left,b=unary();left=v->a.eval(v)*b.eval(v);}else if(eat('/')){Node a=left,b=unary();left=v->a.eval(v)/b.eval(v);}else if(eat('%')){Node a=left,b=unary();left=v->a.eval(v)%b.eval(v);}else if(implicit()){Node a=left,b=unary();left=v->a.eval(v)*b.eval(v);}else return left;}}
    private Node unary(){if(eat('+'))return unary();if(eat('-')){Node n=unary();return v->-n.eval(v);}return power();}
    private Node power(){Node base=postfix();if(eat('^')){Node a=base,b=unary();return v->Math.pow(a.eval(v),b.eval(v));}return base;}
    private Node postfix(){Node value=primary();while(eat('!')){Node n=value;value=v->factorial(n.eval(v));}return value;}
    private Node primary(){spaces();if(eat('(')){Node n=conditional();expect(')');return n;}if(position<source.length()&&(Character.isDigit(source.charAt(position))||source.charAt(position)=='.')){int start=position;while(position<source.length()&&(Character.isDigit(source.charAt(position))||source.charAt(position)=='.'))position++;if(position<source.length()&&source.charAt(position)=='e'){int exponent=position++;if(position<source.length()&&(source.charAt(position)=='+'||source.charAt(position)=='-'))position++;int digits=position;while(position<source.length()&&Character.isDigit(source.charAt(position)))position++;if(digits==position)position=exponent;}final double number;try{number=Double.parseDouble(source.substring(start,position));}catch(NumberFormatException e){throw error("Invalid number");}return v->number;}if(position<source.length()&&Character.isLetter(source.charAt(position))){int start=position;while(position<source.length()&&(Character.isLetterOrDigit(source.charAt(position))||source.charAt(position)=='_'))position++;String name=source.substring(start,position);if(name.equals("pi"))return v->Math.PI;if(name.equals("e"))return v->Math.E;if(name.equals("true"))return v->1;if(name.equals("false"))return v->0;spaces();if(position>=source.length()||source.charAt(position)!='(')return v->v.value(name);position++;Node a=conditional(),b=null;if(eat(','))b=conditional();expect(')');return function(name,a,b);}throw error("Expected a number, variable, function, or '('");}
    private Node function(String n,Node a,Node b){switch(n){case"sin":return v->Math.sin(a.eval(v));case"cos":return v->Math.cos(a.eval(v));case"tan":return v->Math.tan(a.eval(v));case"asin":return v->Math.asin(a.eval(v));case"acos":return v->Math.acos(a.eval(v));case"atan":return v->Math.atan(a.eval(v));case"sinh":return v->Math.sinh(a.eval(v));case"cosh":return v->Math.cosh(a.eval(v));case"tanh":return v->Math.tanh(a.eval(v));case"sqrt":return v->Math.sqrt(a.eval(v));case"cbrt":return v->Math.cbrt(a.eval(v));case"abs":return v->Math.abs(a.eval(v));case"ln":return v->Math.log(a.eval(v));case"log":return v->Math.log10(a.eval(v));case"exp":return v->Math.exp(a.eval(v));case"floor":return v->Math.floor(a.eval(v));case"ceil":return v->Math.ceil(a.eval(v));case"round":return v->Math.rint(a.eval(v));case"sign":return v->Math.signum(a.eval(v));case"deg":return v->Math.toDegrees(a.eval(v));case"rad":return v->Math.toRadians(a.eval(v));case"sec":return v->1/Math.cos(a.eval(v));case"csc":return v->1/Math.sin(a.eval(v));case"cot":return v->1/Math.tan(a.eval(v));case"min":second(n,b);return v->Math.min(a.eval(v),b.eval(v));case"max":second(n,b);return v->Math.max(a.eval(v),b.eval(v));case"pow":second(n,b);return v->Math.pow(a.eval(v),b.eval(v));case"mod":second(n,b);return v->a.eval(v)%b.eval(v);case"hypot":second(n,b);return v->Math.hypot(a.eval(v),b.eval(v));case"atan2":second(n,b);return v->Math.atan2(a.eval(v),b.eval(v));default:throw error("Unknown function '"+n+"'");}}
    private void second(String name,Node b){if(b==null)throw error(name+" needs two values");}
    private String comparisonOperator(){spaces();for(String op:new String[]{"<=",">=","==","!=","<",">"})if(source.startsWith(op,position)){position+=op.length();return op;}return null;}
    private boolean implicit(){spaces();if(position>=source.length())return false;char c=source.charAt(position);return c=='('||c=='.'||Character.isLetterOrDigit(c);}
    private boolean eat(char c){spaces();if(position<source.length()&&source.charAt(position)==c){position++;return true;}return false;}
    private void expect(char c){if(!eat(c))throw error("Expected '"+c+"'");}
    private void spaces(){while(position<source.length()&&Character.isWhitespace(source.charAt(position)))position++;}
    private ParseException error(String message){return new ParseException(message+" at position "+(position+1));}
    private static boolean truth(double value){return Double.isFinite(value)&&Math.abs(value)>1e-12;}
    private static double factorial(double value){int n=(int)Math.rint(value);if(n<0||n>170||Math.abs(value-n)>1e-9)return Double.NaN;double result=1;for(int i=2;i<=n;i++)result*=i;return result;}
}
```
