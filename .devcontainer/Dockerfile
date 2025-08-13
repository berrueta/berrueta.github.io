# Use official Ruby image as base
FROM mcr.microsoft.com/devcontainers/ruby:3.2-bullseye

# Install Jekyll and Bundler
RUN gem install jekyll bundler

# Set working directory
WORKDIR /workspaces/berrueta.github.io

# Expose Jekyll default port
EXPOSE 4000
