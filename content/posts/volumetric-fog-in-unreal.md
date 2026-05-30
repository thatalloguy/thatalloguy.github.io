---
title: "Custom Volumetric Fog in Unreal 5"
date: 2026-05-30
draft: false
tags: ["graphics", "unreal"]
categories: ["graphics", "unreal"]
---




Here at Buas I've been working with a really talented team to make a really cool game. One of the more interesting features I have been personally working on is Volumetric fog for our game.


![CoolGame](/images/fog/coolgame.png)
*Note: this is an screenshot of early on in the development, final game might look different.*

---

You see, each programmer had too pick their own research topic.
Now I have already given a presentation on how I did it and what problems I encountered. However that presentation didn't go into much depth of how its actually coded and was more theory than actual code.


# Setup
First of all our game is a blueprint Unreal project, which means that anything we want to do in C++ is a plugin.
Which means that the Fog is also a C++ plugin. You can create your own plugins directly inside of the Unreal Editor!

The Fog plugin (Named Le Sorcier) has 2 Sides: CPU-side and GPU-side. 
The CPU side just gathers all of the needed data and then transfers it to the GPU-side, 
where the shader/rendering logic actually runs.

---

The Fog pipeline hosts a total of 4 (compute) shaders:

![RenderPipeline](/images/fog/pipeline.png)

**FogMarchShader:**
Here we do raymarching to actually render the volumetric fog.

**Composite:**
Since our fog can be rendered at a different resolution than the actual resolution, we have a composite pass to compose the output of the Fog correctly into the final frame.

**Splat pass:** 
In order to keep track of Actors that can interact with the fog, we splat their **Capsule shape** to a 3d volume texture.

**Decay pass**
In order to have the fog slowly return to its original shape after an actor walking though it we have a decay pass. Here we just slowly fade the 3d Volume texture back to empty.

---

# CPU side
The CPU side can be divided into 3 sections:
- The Shader(s) (input definitions). 
- The View extension.
- The Actor/Components



First we have *Le_SorcierShaders.h* in which I define all of the inputs each shader requires.
For each shader we define a new class that inherits from *FGlobalShader* and inside we define all of the parameters we need. For example here is a snippet of the *FDustFogMarchCS*:

```c++

class FDustFogMarchCS : public FGlobalShader
{
    DECLARE_GLOBAL_SHADER(FDustFogMarchCS);
    SHADER_USE_PARAMETER_STRUCT(FDustFogMarchCS, FGlobalShader);

    static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
    {
        return true;
    }

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters,)
        //Light data
        SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<FLeSorcierLightGPU>, Lights)
        SHADER_PARAMETER(uint32, NumLights)
        SHADER_PARAMETER(float, Anisotropy)

        ///All of the other parameters....
        ///...
    
    END_SHADER_PARAMETER_STRUCT()
};


```

Now that we have defined what data we will be using inside of the shader we can now get the data and set it up for the shader. We do this in a new class that inherits from Unreal's *FSceneViewExtensionBase*.
```c++

class FLeSorcierViewExtension : public FSceneViewExtensionBase
{
public:
	FLeSorcierViewExtension(const FAutoRegister& AutoRegister);

	virtual void SetupViewFamily(FSceneViewFamily& InViewFamily) override;
	virtual void SetupView(FSceneViewFamily& InViewFamily, FSceneView& InView) override {}
	virtual void BeginRenderViewFamily(FSceneViewFamily& InViewFamily) override {}

	virtual void PrePostProcessPass_RenderThread(
		FRDGBuilder& GraphBuilder,
		const FSceneView& View,
		const FPostProcessingInputs& Inputs) override;

private:
	struct FSnapshot
	{
        //...
	};

	FSnapshot Snapshot;
	
	TArray<FLeSorcierLightGPU> SnapshotLights;
	TArray<FLeSorcierInfluencerGPU> SnapshotInfluencers;
	TRefCountPtr<IPooledRenderTarget> VolumeRT;
	
	FTextureRHIRef SnapshotNoiseTextureRHI;
	FTextureRHIRef SnapshotBlueNoiseTextureRHI;
};

```

The main code lives inside of the *SetupViewFamily* and the *PrePostProcessPass_RenderThread*.
With the SetupViewFamily gathering all of the data we need from the scene and the PrePostProcessPass uploading everything to the shader.

From our scene we need a bit of data. We store our fog settings in an Actor, so that it can be tweaked on a per-level basis. It holds no logic whatsoever, its purely there to allow people to edit the fog from inside of the editor.

```c++
//Note these aren't all of the settings, but just a couple to give you the idea.

UCLASS(ClassGroup = "LeSorcier", hidecategories = (Input, Collision, /*other categories*/))
class LE_SORCIER_API ALeSorcierFogActor : public AInfo
{
    GENERATED_BODY()

public:
    ALeSorcierFogActor();

    UPROPERTY(EditAnywhere, Category = "LeSorcier|General")
    bool bEnabled = true;

    UPROPERTY(EditAnywhere, Category = "LeSorcier|Volume")
    FVector VolumeExtent = FVector(6400.0, 6400.0, 6400.0);
    
    UPROPERTY(EditAnywhere, Category = "LeSorcier|Cloud")
    TSoftObjectPtr<UVolumeTexture> NoiseTexture;
 
    
    UPROPERTY(EditAnywhere, Category = "LeSorcier|DistanceFog", meta = (ClampMin = "0"))
    float Density = 0.05f;

};
```

<br>
<br>

First thing to note is that we gather "Snapshots" of everything, this is because some data can't be directly uploaded to the shader.
```c
void FLeSorcierViewExtension::SetupViewFamily(FSceneViewFamily& InViewFamily)
{
    FSceneViewExtensionBase::SetupViewFamily(InViewFamily);
    
    //FSnapShot is basically just a copy of ALeSorcierFogActor, but without being an actual actor.
    Snapshot = FSnapshot();

    SnapshotLights.Reset();
    SnapshotInfluencers.Reset();
    

    ...
}
```

Secondly we check if the scene holds a FogActor and if its enabled.
``` c
void FLeSorcierViewExtension::SetupViewFamily(FSceneViewFamily& InViewFamily)
{
    ...


    UWorld* World = InViewFamily.Scene ? InViewFamily.Scene->GetWorld() : nullptr;
    if (!World) return;

    ALeSorcierFogActor* FogActor = nullptr;
    for (TActorIterator<ALeSorcierFogActor> It(World); It; ++It)
    {
        FogActor = *It;
        break;
    }

    if (!FogActor || !FogActor->bEnabled) return;

    //Now update the SnapShot struct to reflect the data inside of the FogActor.

    ...
}
```

<br>
After we have gathered all of the data from the Actors and Components that we need, we gather all of the lighting data. In my case thats only the point lights, for which I save the following:

```c++
struct FLeSorcierLightGPU
{
    FVector3f Position;
    float     Radius;
    FVector3f Color;
    float     FalloffExponent;
};
```
```c
    for (TActorIterator<APointLight> It(World); It; ++It)
    {
        UPointLightComponent* Comp = It->PointLightComponent;
        //Only add Visible and if they dont have the NoFog tag     
        if (!Comp || !Comp->IsVisible() || It->ActorHasTag(LeSorcierNoFogTag)) continue;

        FLeSorcierLightGPU L;

        L.Position = FVector3f(Comp->GetComponentLocation());
        
        const FLinearColor C = Comp->GetLightColor() * Comp->Intensity;
        
        L.Color = FVector3f(C.R, C.G, C.B);
        L.Radius = Comp->AttenuationRadius;
        L.FalloffExponent = Comp->LightFalloffExponent;
        
        SnapshotLights.Add(L);
    }
```

<br>

Finally its time to actually setup the parameters and dispatch the shaders. Its really not that difficult since we already did most of the heavy lifting. It sometimes is tricky, because you can forgot a parameter, but it should be fine

```c

void FLeSorcierViewExtension::PrePostProcessPass_RenderThread(
    FRDGBuilder& GraphBuilder,
    const FSceneView& View,
    const FPostProcessingInputs& Inputs)
{
    {...}

    FDustFogMarchCS::FParameters* Params =
        GraphBuilder.AllocParameters<FDustFogMarchCS::FParameters>();

    Params->CameraPos      = FVector3f(VM.GetViewOrigin());
    Params->ScreenToWorld  = FMatrix44f(InvViewProj);
    Params->Enabled        = Snapshot.bEnabled;
    Params->Density        = Snapshot.Density;
    Params->StepCount      = Snapshot.StepCount;
    Params->MaxDistance    = Snapshot.MaxDistance;
    //Do this for every other parameter.

    {...}

}
```

Finally its time to dispatch, which I as the following:
```c
    TShaderMapRef<FDustFogMarchCS> MarchCS(GetGlobalShaderMap(View.GetFeatureLevel()));
    FComputeShaderUtils::AddPass(
        GraphBuilder,
        RDG_EVENT_NAME("LeSorcier.DustFogMarch"),
        MarchCS, Params,
        FComputeShaderUtils::GetGroupCount(RenderSize, FIntPoint(8, 8)));
```

---

# GPU Side
Now that we have done all of that tedious setup work, we can finally do the fun shader part 🥳.
For my sanity's sake I will skip over the splat and decay pass.

First we calculate the UV, Depth and WorldPosition

```hlsl

float3 ReconstructWorldPos(float2 UV, float DeviceZ)
{
    float2 NDC = float2(UV.x * 2.0 - 1.0, 1.0 - UV.y * 2.0);
    float4 ClipPos = float4(NDC, DeviceZ, 1.0);
    float4 WorldPos = mul(ClipPos, ScreenToWorld);
    return WorldPos.xyz / WorldPos.w;
}



void MainCS(uint3 DispatchThreadId : SV_DispatchThreadID) 
{

    uint2 HalfPixel = DispatchThreadId.xy;
    uint2 HalfSize = uint2(ceil(float2(RenderRectSize) / DownscaleFactor));
    if (any(HalfPixel >= HalfSize)) return;
    
    //Early out, if we aren't enabled don't waste time on heavy math.
    if (!Enabled)
    {
        OutputTexture[HalfPixel] = float4(0, 0, 0, 1);
        return;
    }
    
    
    float2 FullLocal = (float2(HalfPixel) + 0.5) * DownscaleFactor;
    float2 ViewportUV = FullLocal / float2(RenderRectSize);
    float2 SampleUV = (float2(RenderRectMin) + FullLocal) * BufferInvSize;

    
    float Depth = SceneDepthTexture.SampleLevel(PointSampler, SampleUV, 0).r;
    float3 WorldPos = ReconstructWorldPos(ViewportUV, Depth);
}
```

<br>
<br>
I first apply a layer of distance based fog, this is to add a bit of fog to the entire scene.

```hlsl
    //MaxDistance, CameraPos and Denisity are parameters

    float Distance = length(WorldPos - CameraPos);
    float DistFog  = saturate(Distance / MaxDistance) * Density * 20.0; // 20 = magic number bad.

    //Note: this is just an example not actually in the final code.
    float4 FinalColor = float4(0, 0, 0, 1);
    FinalColor.rgb = DistFog * FogColor; // Fog color is a parameter.
```

Even this simple little math already brings some nice results.


![RenderPipeline](/images/fog/distance.png)

After the distance I add a "cloud" layer, which is the real sauce.

```hlsl

    float DepthForDir = (Depth < 0.0001) ? 0.5 : Depth;
    float3 RayTarget = ReconstructWorldPos(ViewportUV, DepthForDir);
    float3 rayDir = normalize(RayTarget - CameraPos);
    float rayLength = (Depth < 0.0001) ? MaxCloudDistance
                                          : min(length(RayTarget - CameraPos), MaxCloudDistance);
    float stepSize = rayLength / StepCount;

    float  transmittance = 1.0;
    float3 scatterLight  = 0;
    
    ///NOTE: if you are smarter than me, make this not hard-coded and editable for artists.
    const float LightIntensityScale = 500.0;

    //little bit of jitter to avoid repetition and should be picked up nicely by Unreal's TAA.
    float jitter = InterleavedGradientNoise(float2(HalfPixel), FrameIndex % 8);

    ///Main raymarching loop
    for (int i = 0; i < StepCount; ++i)
    {
        //early out, because it will be basically invisible anyways.
        if (transmittance < 0.02) break;

        float t = (float(i) + jitter) * stepSize;
        float3 samplePos = CameraPos + rayDir * t;
        float density = SampleWorldPos(samplePos); // SampleWorldPos is where the magic happends.

        if (density < 1e-4) continue;

        float stepT = exp(-density * stepSize);

        float3 inScatter = 0;
        //Calculate how much lighting should be applied
        for (uint l = 0; l < NumLights; l++)
        {
            FLightGPU light = Lights[l];

            float3 toLight = light.Position - samplePos;
            float dist = length(toLight);
            float fallof = GetFallOf(dist, light.Radius, light.FalloffExponent);
            float cosTheta = dot(toLight, rayDir) / max(dist, 1e-4);
            float phase = phaseHG(cosTheta, Anisotropy);

            inScatter += light.Color * fallof * phase * LightIntensityScale;
        }

        scatterLight  += transmittance * (1 - stepT) * inScatter * FogColor;
        transmittance *= stepT;
    }    
```

This looks relatively simple, because it is! The real magic is happening in the SampleWorldPos function, which goes as following.

```hlsl

float Noise(float3 pointInSpace)
{
    return NoiseTexture.SampleLevel(NoiseSampler, pointInSpace, 0).r;
}

//todo optimize this.
float SampleWorldPos(float3 worldPos)
{
    float heightMask = 1.0 - abs(worldPos.z - CloudCenterZ) / CloudThickness;
    if (heightMask <= 0.0) return 0.0;
    heightMask = saturate(heightMask);

    float clearFactor = 1.0;
    float3 localWind = 0;

    //calculate the wind amount + direction
    {
        float3 VolumeMin = VolumeCenter - VolumeExtentWS * 0.5;
        float3 VolumeMax = VolumeCenter + VolumeExtentWS * 0.5;

        if (all(worldPos >= VolumeMin) && all(worldPos <= VolumeMax))
        {
            float3 volUV = (worldPos - VolumeCenter) / VolumeExtentWS + 0.5;
            float4 vol = VelocityVolume.SampleLevel(VolumeSampler, volUV, 0);
            clearFactor = 1.0 - saturate(vol.a * VolumeClearStrength);
            if (clearFactor < 0.01) return 0.0;
            localWind = vol.rgb * vol.a * VolumePushStrength;
        }
    }
    
    float3 p = worldPos * NoiseScale + WindOffset * Time + localWind;

    float3 pHalf = p * 0.5;
    float3 warp;
    
    warp.x = Noise(pHalf + float3(Time * 0.01, 0, 0));
    warp.y = Noise(pHalf + float3(0, Time * 0.02, 0));
    warp.z = Noise(pHalf + float3(0, 0, Time * 0.03));
    p += (warp - 0.5);

    float distRatio = saturate(length(worldPos - CameraPos) / MaxCloudDistance);

    float base = Noise(p);

    //Use if-statements here to avoid using more Noise() than needed, because its expensive.
    if (distRatio < 0.75)
    {
        base += Noise(p * 2.0) * 0.5;
        if (distRatio < 0.4)
        {
            base += Noise(p * 4.0) * 0.25;
            base *= (1.0 / 1.75);
        }
        else
        {
            base *= (1.0 / 1.5);
        }
    }

    float shape = saturate((base - CloudCoverage) / (1.0 - CloudCoverage));
    return shape * heightMask * CloudStrength * clearFactor;
}


```
This produces something along the lines of this:
![RenderPipeline](/images/fog/cloud.png)

<br>

Finally i get the final color as the following:

```hlsl

    float distT = 1.0 - DistFog;
    float finalT = transmittance * distT;

    //I do " * (1.0 - finalT)" to disable the clouds behind any translucent objects.
    float3 fogRgb = FogColor * (1.0 - finalT) + scatterLight;

    OutputTexture[HalfPixel] = float4(fogRgb, finalT);

```

---

# Tips

I learned many thins while developing the fog, so I wanted to leave some tips for anyone trying to do something similar:

## Materials
Unreal engine support many different kinds of materials so please keep them in mind while creating your fog.
So materials such as Subsurface Scattered ones are directly encoded in the ColorBuffer you get. 
Some aren't like Emissive materials and will require specific solutions to solve. (For emissive materials i just added a new component that would make the fog treat the object as if it were a point light).

## Translucency
Translucency can be a big hassle and something even Unreal's local fog volumes don't fix. Its a problem because you want your fog to stop in the volume, but continue after. This means that you will need to know when a point in space is in a translucent object or not, which is very difficult.

## Performance
One thing I haven't mentioned yet, but performance can be tricky. Especially if you do many steps and many noise samples. 
You can try downscaling your fog like I did or lowering the amount of steps you do, but It can look noisy very quickly. 

## Froxels (Frustum voxels)
Not mentioned here, but Froxels can potentially fix the translucency and emission problem. 
In fact, many triple-A games use this solution (Such as Red dead redemption 2 & The last of us part 2), with also Unreal's built in **Volumetric fog** using it.

## Use Actors as Config
Please don't be as stubborn as me and just use Unreal actors as a way to get your settings reflected in the editor.
Its nice and you are making great use of Unreal's built-in serialization and reflection systems. 
It also means that config are per level and not uniformly.

---

# Going beyond
This isn't all of what my current implementation has to offer. As mentioned previously my fog is also interactive.
It can move out of the way when the player walks through it and slowly returns to its original shape.
Its a fun challenge if you got the time, but I wont end up using it because it doesn't really fit into the game.


<video controls width="100%">
  <source src="/videos/fog1.mp4" type="video/mp4">
</video>

---

<br>

<br>

Thank you for reading, don't fog it up!
or maybe do fog it up 🤔