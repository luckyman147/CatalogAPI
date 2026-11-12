FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080
USER app
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

COPY ["src/ProductCatalog.Api/ProductCatalog.Api.csproj","ProductCatalog.Api/"]
RUN dotnet restore "ProductCatalog.Api/ProductCatalog.Api.csproj"
# Copie du reste des sources
COPY src/ProductCatalog.Api/ ProductCatalog.Api/
WORKDIR "/src/ProductCatalog.Api"
# Compilation
RUN dotnet build "ProductCatalog.Api.csproj" -c Release

FROM build AS publish
RUN dotnet publish "ProductCatalog.Api.csproj" -c Release -o /app/publish
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "ProductCatalog.Api.dll"]