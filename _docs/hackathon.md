---
title: Getting hackathon ready
tags:
 - jekyll
 - github
description: Documentation for the hackathon
---


# hackathon

## Introduction to GitHub Pages and Jekyll

This is a simple guide to get you started with Github Pages and Jekyll. For this we will focus mainly on the following:

- What is GitHub Pages?
- What is Jekyll?
- How does Jekyll work?
- Liquid and Front Matter
- How to create a GitHub Pages site with Jekyll


## What is GitHub Pages?

GitHub Pages is a static site hosting service that takes HTML, CSS, and JavaScript files straight from a repository on GitHub, optionally runs the files through a build process, and publishes a website. You can see examples of GitHub Pages sites by checking out the GitHub Pages Showcase. Note: GitHub Pages sites are subject to the following usage limits:

- GitHub Pages source repositories have a recommended limit of 1GB.
- GitHub Pages sites have a soft bandwidth limit of 100GB per month.
- Should not be used to build a commercial site

## What is Jekyll?

Jekyll is a static site generator that's perfect for personal, project, or organization sites. With Jekyll, you can create a simple blog, portfolio, or resume site. You can also use Jekyll to create a more complex site, like a documentation site. Jekyll is a great way to create a site that's easy to maintain and update.


## How does jekyll work?

Jekyll is a static site generator that takes a collection of text files and turns them into a website. Jekyll does this by combining the text files with a layout. The layout is a template that gives the site its basic structure. Jekyll also uses a configuration file to determine how to build the site. The configuration file tells Jekyll where to find the text files, where to find the layout, and where to put the finished website. Jekyll also uses a special folder called `_site` to store the finished website. When you run Jekyll, it takes the text files, combines them with the layout, and puts the finished website in the `_site` folder. You can then copy the contents of the `_site` folder to a web server to make the site live.

## What is a static site generator?

A static site generator is a tool that takes a collection of text files and turns them into a website. The text files are written in a markup language like Markdown or HTML. The static site generator combines the text files with a layout to create the website. The layout is a template that gives the site its basic structure. The static site generator also uses a configuration file to determine how to build the site. The configuration file tells the static site generator where to find the text files, where to find the layout, and where to put the finished website. The static site generator also uses a special folder to store the finished website. When you run the static site generator, it takes the text files, combines them with the layout, and puts the finished website in the special folder. You can then copy the contents of the special folder to a web server to make the site live.

## Liquid and Front Matter

- Liquid is a templating language that Jekyll uses to process templates. Liquid is a simple language that allows you to insert variables, loops, and conditionals into your templates. Liquid is used to generate the site. Liquid is not displayed on the site. Liquid is a way to generate a site from a set of text files and a layout.

- Front Matter is a block of YAML or JSON at the beginning of a file that is used to set variables for the file. Front Matter is used to set variables like the title of a page, the layout of a page, and the date of a page. Front Matter is processed by Jekyll and used to generate the site. Front Matter is not displayed on the site. Front Matter is a way to set variables for a file that are used by Jekyll to generate the site.

   ```markdown
   ---
   layout: post
   title:  "Example Jenkinsfile for publishing cookbooks to Artifactory"
   date:   2021-06-22 09:35:12 -0500
   categories: Demo Jenkinsfile
   ---
   ```

## How to create a GitHub Pages site with Jekyll

1. Create a new repository on GitHub, name it `<your-github-username>.github.io`. To avoid errors, do not initialize the new repository with README, license, or gitignore files. You can add these files after your project has been pushed to GitHub.

2. Open Terminal.

3. Change to the directory where you want to store your project.

4. Run the following command:

```bash
git clone
```

5. Go to the directory where you cloned your repository.

6. Run the following command:

```bash
jekyll new .
```

7. Run the following command:

```bash
bundle install
```

8. Run the following command:

```bash
bundle exec jekyll serve
```

9. Go to `http://localhost:4000` in your browser to see your site.

10. Make changes to your site and see them reflected in your browser.

11. When you're happy with your site, stop the Jekyll server by pressing `Ctrl+C` in your terminal.

12. Run the following command:

```bash
git add .
```

13. Run the following command:

```bash
git commit -m "Initial commit"
```

14. Run the following command:

```bash
git push
```

15. Go to `https://<your-github-username>.github.io/<your-repository-name>` in your browser to see your site.

16. Share your site.


__**Note:**__ If you want to do the majority of the setup directly in github, it can be done in the repo settings. Create the repository, name it `<your-github-username>.github.io`, and then go to the settings tab. Scroll down to the GitHub Pages section and select the `main` branch as the source. Select github actions as the controller and jekyll as the framerwork. This will genrate a basic configuration file for you. You can then clone the repository and follow the steps above to get started.
