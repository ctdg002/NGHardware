
##### c++可选
+ Transmittance2Sun = TransmittanceAtGround(AtmosphereLightDir)
   + WorldPos = CameraPosWS
   + AzimuthElevation = (方位角，天顶角)，clamp天顶角
   + WorldDir = (cos(E),0,sin(E)) 太阳轨道地球横切面的太阳方向
   + OpticalDepthRGB = OpticalDepth(WorldPos,WorldDir) = 光路深度
   + 计算透光比
+ TransmittanceColor = Transmittance2Sun/TransmittanceAtZenith
##### shader
SkyAtmosphere.usf