<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
	 

  </PropertyGroup>

  <PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Debug|AnyCPU'">
  <OutputPath>bin\Debug\</OutputPath>
    <Optimize>True</Optimize>
	<DebugType>full</DebugType>
	  <WarningLevel>3</WarningLevel>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Devon4Net.Infrastructure.FluentValidation" Version="8.0.2" />
    <PackageReference Include="EntityFramework" Version="6.5.1" />
    <PackageReference Include="Fluent.Infrastructure" Version="2.0.0-beta-01" />
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="9.0.4" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="9.0.4" />
    <PackageReference Include="Optivem.Framework.Infrastructure.FluentValidation" Version="1.0.23" />
    <PackageReference Include="System.Data.SqlClient" Version="4.9.0" />
  </ItemGroup>

</Project>
