## Controller 
/Users/tennetdepositoryfullstack1/go/pkg/mod/github.com/!go!admin!group/go-admin@v1.2.23/plugins/admin/controller


- [ApiCreate](#api_creatego--apicreate)
- [ApiCreateForm](#api_creatego--apicreateform)
- [ApiDetail](#api_detailgo--apidetail)
- [ApiList](#api_listgo--apilist)
- [ApiUpdate](#api_updatego--apiupdate)
- [ApiUpdateForm](#api_updatego--apiupdateform)
- [Auth]
- [Logout]
- [ShowLogin] 
- [TestInfoUrl]
- [TestIsNewUrl]
- [Delete] 
- [ShowDetail]
- [ShowForm]
- [EditForm]
- [GlobalDetaultHandler] 
- [ShowInstall] 
- [CheckDatabase] 
- [ShowMenu] 
- [ShowNewMenu] 
- [ShowEditMenu] 
- [DeleteMenu] 
- [EditMenu] 
- [NewMenu] 
- [MenuOrder] 
- [ShowNewForm] 
- [NewForm] 
- [Operation] 
- [RecordOperationLog]
- [GetPluginsPageJS] 
- [Plugins] 
- [PluginStore] 
- [PluginDetail] 
- [GetPluginBoxParamFromPlug]
- [PluginDownload] 
- [ServerLogin] 
- [ShowInfo] 
- [Assets] 
- [Export] 
- [SystemInfo] 
- [Update] 


-----------------------------------
### Controller API

##### api_create.go > ApiCreate
``` shell
# Learning
func (h *Handler) ApiCreate(ctx *context.Context) {
	param := guard.GetNewFormParam(ctx)

	if len(param.MultiForm.File) > 0 {
		err := file.GetFileEngine(h.config.FileUploadEngine.Name).Upload(param.MultiForm)
		if err != nil {
			response.Error(ctx, err.Error())
			return
		}
	}

	err := param.Panel.InsertData(param.Value())
	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	response.Ok(ctx)
}
```

##### api_create.go > ApiCreateForm
``` shell
# Learning
func (h *Handler) ApiCreateForm(ctx *context.Context) {

	var (
		params           = guard.GetShowNewFormParam(ctx)
		prefix, paramStr = params.Prefix, params.Param.GetRouteParamStr()
		panel            = h.table(prefix, ctx)
		formInfo         = panel.GetNewFormInfo()
		infoUrl          = h.routePathWithPrefix("api_info", prefix) + paramStr
		newUrl           = h.routePathWithPrefix("api_new", prefix)
		referer          = ctx.Referer()
		f                = panel.GetActualNewForm()
	)

	if referer != "" && !isInfoUrl(referer) && !isNewUrl(referer, ctx.Query(constant.PrefixKey)) {
		infoUrl = referer
	}

	response.OkWithData(ctx, map[string]interface{}{
		"panel": formInfo,
		"urls": map[string]string{
			"info": infoUrl,
			"new":  newUrl,
		},
		"pk":     panel.GetPrimaryKey().Name,
		"header": f.HeaderHtml,
		"footer": f.FooterHtml,
		"prefix": h.config.PrefixFixSlash(),
		"token":  h.authSrv().AddToken(),
		"operation_footer": formFooter("new", f.IsHideContinueEditCheckBox, f.IsHideContinueNewCheckBox,
			f.IsHideResetButton, f.FormNewBtnWord),
	})
}
```

##### api_detail.go > ApiDetail
``` shell
# Learning
func (h *Handler) ApiDetail(ctx *context.Context) {
	prefix := ctx.Query(constant.PrefixKey)
	id := ctx.Query(constant.DetailPKKey)
	panel := h.table(prefix, ctx)
	user := auth.Auth(ctx)

	newPanel := panel.Copy()

	formModel := newPanel.GetForm()

	var fieldList types.FieldList

	if len(panel.GetDetail().FieldList) == 0 {
		fieldList = panel.GetInfo().FieldList
	} else {
		fieldList = panel.GetDetail().FieldList
	}

	formModel.FieldList = make([]types.FormField, len(fieldList))

	for i, field := range fieldList {
		formModel.FieldList[i] = types.FormField{
			Field:        field.Field,
			FieldClass:   field.Field,
			TypeName:     field.TypeName,
			Head:         field.Head,
			Hide:         field.Hide,
			Joins:        field.Joins,
			FormType:     form.Default,
			FieldDisplay: field.FieldDisplay,
		}
	}

	param := parameter.GetParam(ctx.Request.URL,
		panel.GetInfo().DefaultPageSize,
		panel.GetInfo().SortField,
		panel.GetInfo().GetSort())

	paramStr := param.DeleteDetailPk().GetRouteParamStr()

	editUrl := modules.AorEmpty(!panel.GetInfo().IsHideEditButton, h.routePathWithPrefix("show_edit", prefix)+paramStr+
		"&"+constant.EditPKKey+"="+ctx.Query(constant.DetailPKKey))
	deleteUrl := modules.AorEmpty(!panel.GetInfo().IsHideDeleteButton, h.routePathWithPrefix("delete", prefix)+paramStr)
	infoUrl := h.routePathWithPrefix("info", prefix) + paramStr

	editUrl = user.GetCheckPermissionByUrlMethod(editUrl, h.route("show_edit").Method())
	deleteUrl = user.GetCheckPermissionByUrlMethod(deleteUrl, h.route("delete").Method())

	deleteJs := ""

	if deleteUrl != "" {
		deleteJs = fmt.Sprintf(`<script>
function DeletePost(id) {
	swal({
			title: '%s',
			type: "warning",
			showCancelButton: true,
			confirmButtonColor: "#DD6B55",
			confirmButtonText: '%s',
			closeOnConfirm: false,
			cancelButtonText: '%s',
		},
		function () {
			$.ajax({
				method: 'post',
				url: '%s',
				data: {
					id: id
				},
				success: function (data) {
					if (typeof (data) === "string") {
						data = JSON.parse(data);
					}
					if (data.code === 200) {
						location.href = '%s'
					} else {
						swal(data.msg, '', 'error');
					}
				}
			});
		});
}

$('.delete-btn').on('click', function (event) {
	DeletePost(%s)
});

</script>`, language.Get("are you sure to delete"), language.Get("yes"), language.Get("cancel"), deleteUrl, infoUrl, id)
	}

	desc := panel.GetDetail().Description

	if desc == "" {
		desc = panel.GetInfo().Description + language.Get("Detail")
	}

	formInfo, err := newPanel.GetDataWithId(param.WithPKs(id))

	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	response.OkWithData(ctx, map[string]interface{}{
		"panel":    formInfo,
		"previous": infoUrl,
		"footer":   deleteJs,
		"prefix":   h.config.PrefixFixSlash(),
	})
}

```

##### api_list.go > ApiList
``` shell
# Learning
func (h *Handler) ApiList(ctx *context.Context) {
	prefix := ctx.Query(constant.PrefixKey)

	panel := h.table(prefix, ctx)

	params := parameter.GetParam(ctx.Request.URL, panel.GetInfo().DefaultPageSize, panel.GetInfo().SortField,
		panel.GetInfo().GetSort())

	panel, panelInfo, urls, err := h.showTableData(ctx, prefix, params, panel, "api_")
	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	response.OkWithData(ctx, map[string]interface{}{
		"panel":  panelInfo,
		"footer": panelInfo.Paginator.GetContent() + panel.GetInfo().FooterHtml,
		"header": aDataTable().GetDataTableHeader() + panel.GetInfo().HeaderHtml,
		"prefix": h.config.PrefixFixSlash(),
		"urls": map[string]string{
			"edit":   urls[0],
			"new":    urls[1],
			"delete": urls[2],
			"export": urls[3],
			"detail": urls[4],
			"info":   urls[5],
			"update": urls[6],
		},
	})
}
```

##### api_update.go > ApiUpdate
``` shell
# Learning
func (h *Handler) ApiUpdate(ctx *context.Context) {
	param := guard.GetEditFormParam(ctx)

	if len(param.MultiForm.File) > 0 {
		err := file.GetFileEngine(h.config.FileUploadEngine.Name).Upload(param.MultiForm)
		if err != nil {
			response.Error(ctx, err.Error())
			return
		}
	}

	for i := 0; i < len(param.Panel.GetForm().FieldList); i++ {
		if param.Panel.GetForm().FieldList[i].FormType == form.File &&
			len(param.MultiForm.File[param.Panel.GetForm().FieldList[i].Field]) == 0 &&
			param.MultiForm.Value[param.Panel.GetForm().FieldList[i].Field+"__delete_flag"][0] != "1" {
			delete(param.MultiForm.Value, param.Panel.GetForm().FieldList[i].Field)
		}
	}

	err := param.Panel.UpdateData(param.Value())
	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	response.Ok(ctx)
}

```

##### api_update.go > ApiUpdateForm
``` shell
# Learning
func (h *Handler) ApiUpdateForm(ctx *context.Context) {
	params := guard.GetShowFormParam(ctx)

	prefix, param := params.Prefix, params.Param

	panel := h.table(prefix, ctx)

	user := auth.Auth(ctx)

	paramStr := param.GetRouteParamStr()

	newUrl := modules.AorEmpty(panel.GetCanAdd(), h.routePathWithPrefix("api_show_new", prefix)+paramStr)
	footerKind := "edit"
	if newUrl == "" || !user.CheckPermissionByUrlMethod(newUrl, h.route("api_show_new").Method(), url.Values{}) {
		footerKind = "edit_only"
	}

	formInfo, err := panel.GetDataWithId(param)

	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	infoUrl := h.routePathWithPrefix("api_info", prefix) + param.DeleteField(constant.EditPKKey).GetRouteParamStr()
	editUrl := h.routePathWithPrefix("api_edit", prefix)

	f := panel.GetForm()

	response.OkWithData(ctx, map[string]interface{}{
		"panel": formInfo,
		"urls": map[string]string{
			"info": infoUrl,
			"edit": editUrl,
		},
		"pk":     panel.GetPrimaryKey().Name,
		"header": f.HeaderHtml,
		"footer": f.FooterHtml,
		"prefix": h.config.PrefixFixSlash(),
		"token":  h.authSrv().AddToken(),
		"operation_footer": formFooter(footerKind, f.IsHideContinueEditCheckBox, f.IsHideContinueNewCheckBox,
			f.IsHideResetButton, f.FormEditBtnWord),
	})
}
```

##### auth.go > Auth 
``` shell
# Auth check input passsword and username for authentication.
func (h *Handler) Auth(ctx *context.Context) {

	var (
		user     models.UserModel
		ok       bool
		errMsg   = "fail"
		s, exist = h.services.GetOrNot(auth.ServiceKey)
	)

	if capDriver, ok := h.captchaConfig["driver"]; ok {
		capt, ok := captcha.Get(capDriver)

		if ok {
			if !capt.Validate(ctx.FormValue("token")) {
				response.BadRequest(ctx, "wrong captcha")
				return
			}
		}
	}

	if !exist {
		password := ctx.FormValue("password")
		username := ctx.FormValue("username")

		if password == "" || username == "" {
			response.BadRequest(ctx, "wrong password or username")
			return
		}
		user, ok = auth.Check(password, username, h.conn)
	} else {
		user, ok, errMsg = auth.GetService(s).P(ctx)
	}

	if !ok {
		response.BadRequest(ctx, errMsg)
		return
	}

	err := auth.SetCookie(ctx, user, h.conn)

	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	if ref := ctx.Referer(); ref != "" {
		if u, err := url.Parse(ref); err == nil {
			v := u.Query()
			if r := v.Get("ref"); r != "" {
				rr, _ := url.QueryUnescape(r)
				response.OkWithData(ctx, map[string]interface{}{
					"url": rr,
				})
				return
			}
		}
	}

	response.OkWithData(ctx, map[string]interface{}{
		"url": h.config.GetIndexURL(),
	})
}
```

##### auth.go > Logout
``` shell
# Logout delete the cookie
func (h *Handler) Logout(ctx *context.Context) {
	err := auth.DelCookie(ctx, db.GetConnection(h.services))
	if err != nil {
		logger.Error("logout error", err)
	}
	ctx.AddHeader("Location", h.config.Url(config.GetLoginUrl()))
	ctx.SetStatusCode(302)
}
```

##### auth.go > ShowLogin
``` shell
# show the login page
// ShowLogin show the login page.
func (h *Handler) ShowLogin(ctx *context.Context) {

	tmpl, name := template.GetComp("login").GetTemplate()
	buf := new(bytes.Buffer)
	if err := tmpl.ExecuteTemplate(buf, name, struct {
		UrlPrefix string
		Title     string
		Logo      template2.HTML
		CdnUrl    string
		System    types.SystemInfo
	}{
		UrlPrefix: h.config.AssertPrefix(),
		Title:     h.config.LoginTitle,
		Logo:      h.config.LoginLogo,
		System: types.SystemInfo{
			Version: system.Version(),
		},
		CdnUrl: h.config.AssetUrl,
	}); err == nil {
		ctx.HTML(http.StatusOK, buf.String())
	} else {
		logger.Error(err)
		ctx.HTML(http.StatusOK, "parse template error (；′⌒`)")
	}
}
```