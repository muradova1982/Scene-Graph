# Scene-Graph
package kr.ac.kaist.vclab.helloopengl3d;

import android.graphics.Bitmap;
import android.graphics.BitmapFactory;
import android.opengl.GLES20;
import android.opengl.GLSurfaceView;
import android.opengl.Matrix;
import android.util.Log;

import org.apache.commons.io.IOUtils;

import java.io.IOException;
import java.io.InputStream;
import java.nio.charset.Charset;
import java.util.Vector;

import javax.microedition.khronos.egl.EGLConfig;
import javax.microedition.khronos.opengles.GL10;

import kr.ac.kaist.vclab.Object3D.Arcball;
import kr.ac.kaist.vclab.Object3D.Geometries.Square;
import kr.ac.kaist.vclab.Object3D.RigTForm;
import kr.ac.kaist.vclab.Object3D.hierarchicalObject.MyRobot;
import kr.ac.kaist.vclab.Scenegraph.Drawer;
import kr.ac.kaist.vclab.Scenegraph.Node.SgGeometryShapeNode;
import kr.ac.kaist.vclab.Scenegraph.Node.SgRbtNode;
import kr.ac.kaist.vclab.Scenegraph.Node.SgRootNode;
import kr.ac.kaist.vclab.Scenegraph.Visitor.RbtAccumVisitor;
import kr.ac.kaist.vclab.util.Constants;
import kr.ac.kaist.vclab.util.Quaternion;
import kr.ac.kaist.vclab.util.ShaderState;

/**
 * Created by sjjeon on 16. 9. 20.
 */

public class MyGLRenderer implements GLSurfaceView.Renderer {

    private static final String TAG = "MyGLRenderer";
    public static final int MOVEMENT_ROTATION = 0;
    public static final int MOVEMENT_TRANSLATION_XY = 1;
    public static final int MOVEMENT_TRANSLATION_DEPTH = 2;

    public boolean pickingMode = false;

    // -------- Shaders
    static final int g_numShaders = 3, g_numRegularShaders = 2;
    static final int DEFAULT_SHADER = 0;
    static final int PICKING_SHADER = 1;
    public Vector<ShaderState> g_shaderStates;
    public int g_activeShader = 0;

    SgRootNode g_world;
    SgRbtNode g_skyNode, g_groundNode, g_robot1Node, g_rabbitNode;
    MyRobot robotFactory;

    SgRbtNode g_currentCameraNode;
    SgRbtNode g_currentPickedRbtNode;

    public static final int ARCBALL_ON_PICKED = 0;
    public static final int ARCBALL_ON_SKY = 1;
    public static final int EGO_MOTION = 2;
    private int manipMode;

    private int viewWidth;
    private int viewHeight;
    final double RAD_PER_DEG = 0.5 * Constants.CS175_PI/180;


    private Arcball mArcBall;

    public float [] mOrthoMatrix = new float[16];
    private float[] mProjMatrix = new float[16];

    private float[] mLight = new float[3];
    private float[] mLight2 = new float[3];



    float touchedX = -1, touchedY = -1;
    @Override
    public void onSurfaceCreated(GL10 unused, EGLConfig config) {

        manipMode = EGO_MOTION;
        // Set the background frame color
        GLES20.glClearColor(1.0f, 1.0f, 1.0f, 1.0f);
        GLES20.glEnable(GLES20.GL_CULL_FACE);
        GLES20.glEnable(GLES20.GL_DEPTH_TEST);

        mLight = new float[] {2.0f, 3.0f, 14.0f};
        mLight2 = new float[] {-2.0f, -3.0f, -5.0f};

        // Initialize arcball
        mArcBall = new Arcball();
        mArcBall.setColor(1.0f, 0.0f, 0.0f);

        robotFactory = new MyRobot();

        initShaders();
        initScene();

    }



    @Override
    public void onDrawFrame(GL10 unused) {
        GLES20.glUseProgram(g_shaderStates.get(g_activeShader).mProgram);
        // Draw background color
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT | GLES20.GL_DEPTH_BUFFER_BIT);

        drawStuff(g_shaderStates.get(g_activeShader), pickingMode);

    }

    public void updateViewMatrix(){

    }
    public void drawStuff(ShaderState curSS, boolean picking){
        //updateArcballScale();

        final RigTForm eyeRbt = RbtAccumVisitor.getPathAccumRbt(g_world, g_currentCameraNode);
        //final RigTForm eyeRbt = new RigTForm(new float[]{0,0,8}, new Quaternion(0.9f, 0.4f,0.05f,0.15f));

        final RigTForm invEyeRbt = RigTForm.inv(eyeRbt);


        float mL[] = {mLight[0], mLight[1], mLight[2], 1};
        float mL2[] = {mLight2[0], mLight2[1], mLight2[2], 1};
        final float eyeLight1[] = invEyeRbt.multiply(mL);
        final float eyeLight2[] = invEyeRbt.multiply(mL2);


        GLES20.glUniform3f(curSS.mLightHandle, eyeLight1[0], eyeLight1[1], eyeLight1[2]);
        GLES20.glUniform3f(curSS.mLight2Handle, eyeLight2[0], eyeLight2[1], eyeLight2[2]);

        Drawer drawer = new Drawer(invEyeRbt, curSS);
        g_world.accept(drawer);

        mArcBall.draw(curSS, invEyeRbt.multiply(getArcballRbt()));
    }



    public RigTForm moveArcball(float[] prevPos, float[] currentPos){
        RigTForm eyeInverse = RigTForm.inv(RbtAccumVisitor.getPathAccumRbt(g_world, g_currentCameraNode));
        return mArcBall.move(prevPos, currentPos, getArcballRbt(), eyeInverse);

    }

    @Override
    public void onSurfaceChanged(GL10 unused, int width, int height) {
        // Adjust the viewport based on geometry changes,
        // such as screen rotation
        GLES20.glViewport(0, 0, width, height);

        viewWidth = width;
        viewHeight = height;

        final float ratio = (float) viewWidth / viewHeight;
        final float left = -ratio;
        final float right = ratio;

        final float bottom = -1.0f;
        final float top = 1.0f;
        final float near = 1f;
        final float far = 10.0f;
        // Create a new perspective projection matrix. The height will stay the same
        // while the width will vary as per aspect ratio.

        Matrix.frustumM(mProjMatrix, 0, left, right, bottom, top, near, far);
        GLES20.glUniformMatrix4fv(g_shaderStates.get(g_activeShader).mProjMatrixHandle, 1, false, mProjMatrix, 0);
        mArcBall.setViewInfo(viewWidth, viewHeight, mProjMatrix);
    }


    public void initShaders() {
        g_shaderStates = new Vector<>();
        // prepare shaders and OpenGL program
        int vertexShader = MyGLRenderer.loadShaderFromFile(
                GLES20.GL_VERTEX_SHADER, "basic-gl2.vshader");
        int fragmentShader = MyGLRenderer.loadShaderFromFile(
                GLES20.GL_FRAGMENT_SHADER, "diffuse-gl2.fshader");
        g_shaderStates.add(new ShaderState(vertexShader, fragmentShader));

        g_activeShader = DEFAULT_SHADER;
    }

    public void initScene(){
        g_world = new SgRootNode();

        g_skyNode = new SgRbtNode(new RigTForm(new float[]{0f,1f, 10.0f}));

        g_groundNode= new SgRbtNode();
        //public SgGeometryShapeNode(Geometry g, float[] color, float[] translate, float[] eulerAngles, float[] scales)
        g_groundNode.addChild(new SgGeometryShapeNode(new Square(), new float[]{0.3f,0.9f, 0.9f}, new float[]{0,-3,0}, new float[]{90f, 0,0}, new float[]{8,8,8} ));

        g_robot1Node =robotFactory.constructRobot(new float[]{1f, 0f, 0f});
        //Task1: Move the robot to the left
        //Put your code here

        g_world.addChild(g_skyNode);
        g_world.addChild(g_groundNode);
        g_world.addChild(g_robot1Node);

        //Task2: Make a new object on the right side of the robot
        //Put your code here

        g_currentCameraNode = g_skyNode;
    }

    public void resetScene(){
        g_skyNode.setRbt(new RigTForm(new float[]{0f,0f, 0.0f}));
    }

    public void setRobot(){
        g_currentCameraNode = g_robot1Node;

    }

    public int getManipMode() {
        // if nothing is picked or the picked transform is the transfrom we are viewing from
        if (g_currentPickedRbtNode == null || g_currentPickedRbtNode == g_currentCameraNode) {
            if (g_currentCameraNode == g_skyNode )
                return ARCBALL_ON_SKY;
            else
                return EGO_MOTION;
        }
        else
            return ARCBALL_ON_PICKED;
    }

    public boolean shouldUseArcball() {
        return true;
    }

    public void setActiveShader(int id){
        g_activeShader = id;
        pickingMode = g_activeShader == PICKING_SHADER;
    }
    // The translation part of the aux frame either comes from the current
// active object, or is the identity matrix when
    public RigTForm getArcballRbt() {
        RigTForm result = null;
        switch (getManipMode()) {
            case ARCBALL_ON_PICKED:
                result =  RbtAccumVisitor.getPathAccumRbt(g_world, g_currentPickedRbtNode);
                break;
            case ARCBALL_ON_SKY:
                result = new RigTForm();
                break;
            case EGO_MOTION:
                result = RbtAccumVisitor.getPathAccumRbt(g_world, g_currentCameraNode);
                break;
            default:
                throw new RuntimeException("Invalid ManipMode");
        }
        return result;

